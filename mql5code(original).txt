#property copyright "HumanBrain EA V7.32 - Position Management Fixed"
#property link      ""
#property version   "7.33"
#property strict
#property description "EA V7.33 HumanBrain Complete - ADVANCED PARTIAL CLOSING SYSTEM"
#property description "FIXES: 5-min cooldown, same-direction limit, proximity check,"
#property description "M5 ATR for SL/TP, position age timeout, raised entry thresholds"
#property description "Q-Learning (108 states), Markov Chains, 9-Factor Threat, 6-Component Confidence"
#property description "*** V7.32: 11 critical position management bugs fixed (4 major enhancements) ***"
#include <Trade/Trade.mqh>
#include <Trade/PositionInfo.mqh>
//+------------------------------------------------------------------+
//| SECTION 1: ENUMERATIONS                                          |
//+------------------------------------------------------------------+
enum ENUM_RISK_PROFILE
{
   RISK_LOW         = 0,    // Low Risk (0.5% per trade)
   RISK_MEDIUM      = 1,    // Medium Risk (1.0% per trade)
   RISK_MEDIUM_HIGH = 2,    // Medium-High Risk (1.5% per trade)
   RISK_HIGH        = 3,    // High Risk (2.0% per trade)
   RISK_VERY_HIGH   = 4     // Very High Risk (3.0% per trade)
};
enum ENUM_AI_MODE
{
   AI_OFF           = 0,    // AI Disabled
   AI_OVERLAY       = 1,    // AI Overlay (confidence boost)
   AI_FULL          = 2     // AI Full Control
};
enum ENUM_EXECUTION_MODE
{
   MARKET       = 0,   // Immediate market order
   PENDING_STOP = 1    // Place stop pending order
};
enum ENUM_RECOVERY_MODE
{
   RECOVERY_OFF        = 0,
   RECOVERY_AVERAGING  = 1,
   RECOVERY_HEDGING    = 2,
   RECOVERY_GRID       = 3,
   RECOVERY_MARTINGALE = 4
};
enum ENUM_COMBO_RANK_MODE
{
   COMBO_RANK_HEURISTIC = 0,
   COMBO_RANK_ENTROPY_IG = 1,
   COMBO_RANK_HYBRID = 2
};
enum ENUM_MARKET_REGIME
{
   REGIME_UNKNOWN    = 0,   // Unknown
   REGIME_TRENDING   = 1,   // Trending (ADX > 25)
   REGIME_RANGING    = 2,   // Ranging (ADX < 20)
   REGIME_VOLATILE   = 3,   // Volatile (ATR > 1.5x avg)
   REGIME_QUIET      = 4    // Quiet (ATR < 0.7x avg)
};
enum ENUM_THREAT_ZONE
{
   THREAT_GREEN     = 0,    // Green (0-20) - Safe
   THREAT_YELLOW    = 1,    // Yellow (21-40) - Caution
   THREAT_ORANGE    = 2,    // Orange (41-60) - Elevated
   THREAT_RED       = 3,    // Red (61-80) - High Risk
   THREAT_EXTREME   = 4     // Extreme (81-100) - Stop Trading
};
enum ENUM_EA_STATE
{
   STATE_IDLE           = 0,  // No positions, waiting
   STATE_ENTRY_SEEKING  = 1,  // Signal detected, evaluating
   STATE_POSITION_ACTIVE = 2, // Main position open
   STATE_POSITION_REDUCED = 3,// After 50% close
   STATE_RECOVERY_ACTIVE = 4, // Recovery orders placed
   STATE_DRAWDOWN_PROTECT = 5,// Equity protection mode
   STATE_EXTREME_RISK   = 6   // Emergency - closing all
};
enum ENUM_RL_ACTION
{
   RL_FULL_SIZE     = 0,    // Trade with 100% size
   RL_HALF_SIZE     = 1,    // Trade with 50% size
   RL_QUARTER_SIZE  = 2,    // Trade with 25% size
   RL_SKIP_TRADE    = 3     // Skip this trade
};
enum ENUM_MARKOV_STATE
{
   MARKOV_WIN       = 0,    // Last trade was win
   MARKOV_LOSS      = 1,    // Last trade was loss
   MARKOV_EVEN      = 2     // Last trade was breakeven
};
//+------------------------------------------------------------------+
//| SECTION 2: INPUT PARAMETERS                                      |
//+------------------------------------------------------------------+
//--- Expiry Lock
input group    "=== Expiry Lock ==="
input int      INPUT_EXPIRY_YEAR  = 2030;         // Expiry Year (FIXED: Extended)
input int      INPUT_EXPIRY_MONTH = 12;           // Expiry Month
input int      INPUT_EXPIRY_DAY   = 31;           // Expiry Day
//--- Trading Configuration
input group    "=== Trading Configuration ==="
input int      INPUT_MAX_CONCURRENT_TRADES  = 3;  // Max concurrent MAIN trades (set 1 for strict single-slot flip behavior)
input int      INPUT_MAX_CONCURRENT_RECOVERY_TRADES = 3; // Max concurrent recovery/aux trades
input int      INPUT_MAX_SAME_DIRECTION    = 2;  // Max same-direction trades
input int      INPUT_SAME_DIRECTION_BLOCK_SECONDS = 360; // Direction-specific re-entry block in seconds (0=disable)
input double   INPUT_PROXIMITY_POINTS      = 0.0; // Proximity rule disabled (timeout-only pacing)
input int      INPUT_POSITION_AGE_HOURS    = 24;  // Close stale positions after N hours (0=disabled)
input int      INPUT_MAGIC_NUMBER           = 770700; // Magic Number
input int      INPUT_ORDER_COOLDOWN_SECONDS = 300; // Cooldown between orders (seconds) FIXED: 5 minutes to prevent clustering
input ENUM_EXECUTION_MODE INPUT_EXECUTION_MODE = MARKET; // Order execution mode
input int      INPUT_PENDING_STOP_OFFSET_POINTS = 30; // Stop trigger offset from market (points)
input int      INPUT_PENDING_EXPIRY_MINUTES = 60; // Pending stop expiry in minutes
input bool     INPUT_RESET_ALL_PERSISTED_STATE = false; // Delete all persisted symbol+magic files on init
input bool     INPUT_CLOSE_ON_OPPOSITE_SIGNAL = false; // Close previous position when opposite signal appears
input bool     INPUT_STRICT_OPPOSITE_FLIP_MODE = true; // When max main=1: force close/cancel opposite exposure before new opposite entry
input bool     INPUT_MAX_MAIN_HARD_CAP_ON = true; // Hard cap: INPUT_MAX_CONCURRENT_TRADES is absolute entry limit (no adaptive expansion in gating)
input bool     INPUT_FLIP_CANCEL_OPPOSITE_PENDING_ON = true; // On opposite flip, cancel opposite-direction MAIN pending stop orders
input bool     INPUT_FLIP_BYPASS_COOLDOWN_ON = true; // Confirmed opposite flip may bypass global cooldown for replacement entry
input bool     INPUT_ALLOW_ADAPTIVE_MAX_POSITION_EXPANSION = false; // Allow adaptive max-position expansion above input cap when hard-cap is OFF
//--- Risk Management
input group    "=== Risk Management ==="
input ENUM_RISK_PROFILE INPUT_RISK_PROFILE = RISK_MEDIUM; // Risk Profile
input double   INPUT_RISK_PERCENT          = 1.0; // Risk % per trade (0=use profile)
input double   INPUT_MAX_LOT_SIZE          = 5.0; // Maximum lot size
input double   INPUT_MIN_LOT_SIZE          = 0.01;// Minimum lot size
input double   INPUT_MAX_TOTAL_RISK_PERCENT = 5.0;// Max total portfolio risk %
//--- Equity Protection
input group    "=== Equity Protection ==="
input double   INPUT_EQUITY_FLOOR_PERCENT    = 85.0; // Equity floor % (close all below)
input double   INPUT_DAILY_LOSS_LIMIT_PERCENT = 3.0; // Max daily loss %
input int      INPUT_MAX_DAILY_TRADES        = 50;   // Max trades per day (FIXED: Increased from 20)
input int      INPUT_MAX_CONSECUTIVE_LOSSES  = 10;   // Max consecutive losses before pause (FIXED: Increased from 5)
input bool     INPUT_RESET_CONSEC_DAILY      = true; // Reset consecutive losses daily
//--- Extreme Risk Controls (Default OFF)
input group    "=== Extreme Risk Controls (Default OFF) ==="
input bool     INPUT_ENABLE_EXTREME_BY_THREAT = false;
input bool     INPUT_ENABLE_EXTREME_BY_DRAWDOWN = false;
input bool     INPUT_ENABLE_EXTREME_HYSTERESIS_EXIT = false;
input bool     INPUT_ENABLE_DRAWDOWN_PROTECT_STATE = false;
input double   INPUT_EXTREME_ENTER_THREAT = 80.0;
input double   INPUT_EXTREME_ENTER_DRAWDOWN = 10.0;
input double   INPUT_EXTREME_EXIT_THREAT = 70.0;
input double   INPUT_EXTREME_EXIT_DRAWDOWN = 7.0;
input int      INPUT_EXTREME_EXIT_MAX_TOTAL_POSITIONS = 1;
input bool     INPUT_ENABLE_EXTREME_ON_TICK_HANDLER = false;
input bool     INPUT_ENABLE_EXTREME_ON_TICK_EARLY_RETURN = false;
input bool     INPUT_ENABLE_EXTREME_CLOSE_OLDEST = false;
input bool     INPUT_ENABLE_EXTREME_FILTER_SYMBOL = false;
input bool     INPUT_ENABLE_EXTREME_FILTER_MAGIC = false;
input bool     INPUT_ENABLE_EXTREME_THROTTLE = false;
input int      INPUT_EXTREME_CLOSE_INTERVAL_SECONDS = 5;
input int      INPUT_EXTREME_MAX_CLOSES_PER_CALL = 1;
input bool     INPUT_ENABLE_EQUITY_FLOOR_TRIGGER = false;
input bool     INPUT_ENABLE_EQUITY_FLOOR_FORCE_EXTREME_STATE = false;
input bool     INPUT_ENABLE_EQUITY_FLOOR_CLOSE_ALL = false;
input bool     INPUT_ENABLE_EQUITY_FLOOR_RETURN_AFTER_ACTION = false;
input bool     INPUT_ENABLE_CLOSE_ALL_POSITIONS_API = false;
input bool     INPUT_ENABLE_CLOSE_ALL_ONLY_OUR_POSITIONS = false; // RISK: false can close positions not created by this EA
input bool     INPUT_ENABLE_CLOSE_ALL_SYMBOL_FILTER = false;
input bool     INPUT_CLOSE_ALL_SYMBOL_SCOPE_CURRENT = true;
input bool     INPUT_ENABLE_GATE_BLOCK_ON_PROTECTION_STATE = false;
input bool     INPUT_ENABLE_THREAT_HARD_BLOCK = false;
input bool     INPUT_ENABLE_THREAT_EXTREME_ZONE_BLOCK = false;
input bool     INPUT_ENABLE_THREAT_SOFT_LOT_SHRINK = false;
input bool     INPUT_ENABLE_CLOSE_RECOVERY_TIMEOUT = false;
input bool     INPUT_ENABLE_CLOSE_POSITION_AGE_TIMEOUT = false;
input bool     INPUT_ENABLE_CLOSE_HIGH_SPREAD_PROFIT = false;
input bool     INPUT_ENABLE_CLOSE_50PCT_DEFENSIVE = false;
input bool     INPUT_ENABLE_CLOSE_PARTIAL_TP = false;
input bool     INPUT_ENABLE_CLOSE_MULTI_LEVEL_PARTIAL = false;
input bool     INPUT_ENABLE_MODIFY_MOVE_TO_BREAKEVEN = false;
input bool     INPUT_ENABLE_MODIFY_TRAILING_SL = false;
input bool     INPUT_ENABLE_MODIFY_TRAILING_TP = false;
input bool     INPUT_ENABLE_MODIFY_SKIP_LOSS_ON_HIGH_SPREAD = false;
input bool     INPUT_USE_LEGACY_BEHAVIOR_MAPPING = true;
input bool     INPUT_FORCE_NEW_TOGGLES_ONLY = false;
enum ENUM_TOGGLE_RESOLUTION_MODE
{
   TOGGLE_RESOLUTION_MIGRATION = 0, // legacy OR new
   TOGGLE_RESOLUTION_NEW_AUTH  = 1, // new is authoritative
   TOGGLE_RESOLUTION_STRICT_NEW = 2 // legacy mapping ignored + strict mismatch warnings
};
input ENUM_TOGGLE_RESOLUTION_MODE INPUT_TOGGLE_RESOLUTION_MODE = TOGGLE_RESOLUTION_NEW_AUTH;
input bool     INPUT_STRICT_EFFECTIVE_CONFIG_VALIDATION = false;
//--- V7.31 Migration Notes (Toggle Semantics)
// New master/sub-feature toggles default to ON to preserve legacy runtime behavior.
// Existing INPUT_ENABLE_* flags remain backward-compatible umbrella controls.
input group    "=== Master Feature Toggles ==="
input bool     INPUT_TOGGLE_PLACE_ORDERS = true;
input bool     INPUT_TOGGLE_CLOSE_ORDERS = true;
input bool     INPUT_TOGGLE_MODIFY_STOPS = true;
input bool     INPUT_TOGGLE_MODIFY_TPS = true;
input bool     INPUT_TOGGLE_PENDING_ORDERS = true;
input bool     INPUT_TOGGLE_MARKET_ORDERS = true;
input bool     INPUT_PENDING_EXPIRY_CLEANUP_ON = true;

input group    "=== Gate Toggles (Entry / Risk) ==="
input bool     INPUT_GATE_TERMINAL_CONNECTED_ON = true;
input bool     INPUT_GATE_AUTOTRADING_ALLOWED_ON = true;
input bool     INPUT_GATE_SESSION_ON = true;
input bool     INPUT_GATE_SESSION_WINDOW_ON = true;
input bool     INPUT_GATE_COOLDOWN_ON = true;
input bool     INPUT_GATE_MAX_DAILY_TRADES_ON = true;
input bool     INPUT_GATE_DAILY_LOSS_ON = true;
input bool     INPUT_GATE_CONSECUTIVE_LOSS_ON = true;
input bool     INPUT_GATE_SPREAD_ON = true;
input bool     INPUT_GATE_MAX_POSITIONS_ON = true;
input bool     INPUT_GATE_EA_PROTECTION_STATE_ON = true;
input bool     INPUT_GATE_DATA_ANOMALY_KILLSWITCH_ON = true;
input bool     INPUT_GATE_SIGNAL_DETECTION_ON = true;
input bool     INPUT_GATE_MIN_SIGNALS_ON = true;
input bool     INPUT_GATE_MTF_WEIGHTING_ON = true;
input bool     INPUT_GATE_ADX_FILTER_ON = true;
input bool     INPUT_GATE_SAME_DIRECTION_ON = true;
input bool     INPUT_GATE_PROXIMITY_ON = true;
input bool     INPUT_GATE_MTF_ALIGNMENT_ON = true;
input bool     INPUT_GATE_THREAT_HARD_BLOCK_ON = true;
input bool     INPUT_GATE_THREAT_EXTREME_BLOCK_ON = true;
input bool     INPUT_GATE_CONFIDENCE_MIN_ON = true;
input bool     INPUT_GATE_EFFECTIVE_RR_ON = true;

input group    "=== Threat Factor Toggles ==="
input bool     INPUT_THREAT_FACTOR_LOSING_COUNT_ON = true;
input bool     INPUT_THREAT_FACTOR_MAJORITY_LOSING_ON = true;
input bool     INPUT_THREAT_FACTOR_DRAWDOWN_ON = true;
input bool     INPUT_THREAT_FACTOR_CONSECUTIVE_LOSS_ON = true;
input bool     INPUT_THREAT_FACTOR_VOLATILITY_RATIO_ON = true;
input bool     INPUT_THREAT_FACTOR_NEWS_WINDOW_ON = true;
input bool     INPUT_THREAT_FACTOR_RECOVERY_POSITION_ON = true;
input bool     INPUT_THREAT_FACTOR_WIN_STREAK_ON = true;
input bool     INPUT_THREAT_FRIDAY_LATE_PENALTY_ON = true;
input bool     INPUT_THREAT_END_OF_MONTH_PENALTY_ON = true;
input bool     INPUT_THREAT_SOFT_LOT_SHRINK_ON = true;
input bool     INPUT_THREAT_HARD_ENTRY_BLOCK_ON = true;

input group    "=== Lot Sizing Toggles ==="
input bool     INPUT_LOT_BASE_RISK_ON = true;
input bool     INPUT_LOT_RL_SCALING_ON = true;
input bool     INPUT_LOT_ADAPTIVE_MULTIPLIER_ON = true;
input bool     INPUT_LOT_STREAK_BOOST_ON = true;
input bool     INPUT_LOT_HIGH_ADX_BOOST_ON = true;
input bool     INPUT_LOT_RISK_PARITY_CAP_ON = true;
input bool     INPUT_LOT_MARGIN_DOWNSCALE_ON = true;

input group    "=== Execution Path Toggles ==="
input bool     INPUT_EXEC_MARKET_PATH_ON = true;
input bool     INPUT_EXEC_PENDING_PATH_ON = true;
input bool     INPUT_EXEC_PENDING_DUPLICATE_BLOCK_ON = true;
input bool     INPUT_EXEC_PENDING_EXPIRY_ON = true;
input bool     INPUT_EXEC_MARKET_RETRY_ON = true;
input bool     INPUT_EXEC_RECORD_RL_ON_SUBMIT = true;

input group    "=== Close Trigger Toggles ==="
input bool     INPUT_CLOSE_EQUITY_FLOOR_ON = true;
input bool     INPUT_CLOSE_HIGH_SPREAD_PROFIT_ON = true;
input bool     INPUT_CLOSE_50PCT_DEFENSIVE_ON = true;
input bool     INPUT_CLOSE_PARTIAL_TP_ON = true;
input bool     INPUT_CLOSE_MULTI_LEVEL_PARTIAL_ON = true;
input bool     INPUT_CLOSE_AGE_TIMEOUT_ON = true;
input bool     INPUT_CLOSE_RECOVERY_TIMEOUT_ON = true;

input group    "=== Modify SL/TP Toggles ==="
input bool     INPUT_MODIFY_BREAKEVEN_ON = true;
input bool     INPUT_MODIFY_TRAILING_SL_ON = true;
input bool     INPUT_MODIFY_TRAILING_TP_ON = true;
input bool     INPUT_MODIFY_SUPPRESS_ON_HIGH_SPREAD_LOSS_ON = true;
input bool     INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON = true; // WARNING: disable only for diagnostics

input group    "=== Session Sub-Toggles ==="
input bool     INPUT_SESSION_ASIAN_ON = true;
input bool     INPUT_SESSION_LONDON_ON = true;
input bool     INPUT_SESSION_NY_ON = true;
input bool     INPUT_SESSION_ALL_OFF_BLOCK_ENTRIES = true;

input group    "=== Learning / Inference Sub-Toggles ==="
input bool     INPUT_RL_INFERENCE_ON = true;
input bool     INPUT_RL_LEARNING_ON = true;
input bool     INPUT_RL_RECORD_ON = true;
input bool     INPUT_MARKOV_INFERENCE_ON = true;
input bool     INPUT_MARKOV_UPDATE_ON = true;
input bool     INPUT_ML_INFERENCE_ON = true;
input bool     INPUT_ML_RECORD_ON = true;
input bool     INPUT_COMBO_ADAPTIVE_INFERENCE_ON = true;
input bool     INPUT_COMBO_ADAPTIVE_RECORD_ON = true;
input bool     INPUT_AI_QUERY_ON = true;
input bool     INPUT_AI_BLEND_ON = true;
//--- Entry Conditions (FIXED: Relaxed thresholds)
input group    "=== Entry Conditions ==="
input int      INPUT_MIN_SIGNALS       = 3;       // Minimum signals required (FIXED: Raised to prevent false entries)
input double   INPUT_MIN_CONFIDENCE    = 55.0;    // Minimum confidence % (FIXED: Raised for quality entries)
input double   INPUT_MAX_THREAT_ENTRY  = 70.0;    // Max threat for new entry
input int      INPUT_MIN_MTF_SCORE     = 2;       // Minimum MTF alignment score (FIXED: Require at least H1 agreement)
input double   INPUT_MTF_CONSENSUS_VOTE_WEIGHT = 2.0; // Extra bull/bear vote weight when H1/H4/D1 directional consensus aligns
input bool     INPUT_USE_ADX_FILTER    = false;   // Use ADX trend filter (FIXED: Disabled by default)
input double   INPUT_ADX_MIN_THRESHOLD = 15.0;    // ADX minimum for trend (FIXED: Reduced from 20)
input bool     INPUT_ENABLE_HIGH_ADX_RISK_MODE = false; // Priority 1: boost confidence/size only when ADX is strong
input double   INPUT_HIGH_ADX_THRESHOLD = 35.0;   // High ADX threshold for risk mode
input double   INPUT_HIGH_ADX_CONFIDENCE_BOOST = 6.0; // Confidence boost when ADX >= threshold
input double   INPUT_HIGH_ADX_LOT_MULTIPLIER = 1.15;  // Extra lot multiplier in high ADX mode
//--- Stop Loss & Take Profit
input group    "=== Stop Loss & Take Profit ==="
input double   INPUT_SL_ATR_MULTIPLIER = 2.0;     // SL = ATR x this
input double   INPUT_TP_ATR_MULTIPLIER = 3.0;     // TP = ATR x this
input double   INPUT_MIN_SL_POINTS     = 200.0;   // Minimum SL in points (FIXED: 200 pts min for XAUUSD ~$2 SL)
input double   INPUT_MAX_SL_POINTS     = 5000.0;  // Maximum SL in points
input double   INPUT_MIN_TP_POINTS     = 300.0;   // Minimum TP in points (FIXED: 300 pts min for viable R:R)
input double   INPUT_MAX_TP_POINTS     = 10000.0; // Maximum TP in points
//--- 50% Lot Close System
input group    "=== 50% Lot Close System ==="
input bool     INPUT_ENABLE_50PCT_CLOSE      = false; // DISABLED - Use V7.33 new system (was buggy)
input double   INPUT_50PCT_TRIGGER_LOW       = 45.0; // Trigger zone lower bound %
input double   INPUT_50PCT_TRIGGER_HIGH      = 55.0; // Trigger zone upper bound %
input bool     INPUT_CONFIDENCE_BASED_CLOSE  = false; // DISABLED - This caused the 25% bug!
//--- Partial Close at TP%
input group    "=== Partial Close at TP ==="
input bool     INPUT_ENABLE_PARTIAL_CLOSE    = false; // DISABLED - Use V7.33 new system
input double   INPUT_PARTIAL_TP_PERCENT      = 50.0; // Close portion at this % of TP
input double   INPUT_PARTIAL_CLOSE_RATIO     = 0.5;  // Close this fraction of lots
input bool     INPUT_MOVE_BE_AFTER_PARTIAL   = true; // Move SL to breakeven after partial
//--- Trailing Stop
input group    "=== Trailing Stop ==="
input bool     INPUT_ENABLE_TRAILING         = true; // Enable trailing stop
input double   INPUT_TRAIL_ATR_MULTIPLIER    = 1.0;  // Trail distance = ATR x this
input double   INPUT_TRAIL_STEP_POINTS       = 50.0; // Min improvement step (points)
input double   INPUT_TRAIL_ACTIVATION_POINTS = 200.0;// Activate after this profit (points)
input bool     INPUT_ENABLE_TRAILING_TP      = true; // Enable trailing TP logic
input bool     INPUT_ENABLE_HIGH_SPREAD_PROTECT = true; // Enable high-spread protective behavior
input bool     INPUT_CLOSE_PROFIT_ON_HIGH_SPREAD = true; // Close profitable running positions when spread spikes
input double   INPUT_HIGH_SPREAD_CLOSE_PERCENT = 50.0; // Percent of profitable position volume to close on high spread (1..100)
input bool     INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD = true; // Do not adjust losing-position SL/TP during spread spikes
input double   INPUT_HIGH_SPREAD_MULTIPLIER = 5.0; // Spread spike threshold as multiple of rolling average

//+------------------------------------------------------------------+
//| V7.33 NEW: ADVANCED PARTIAL CLOSING SYSTEM                        |
//+------------------------------------------------------------------+
input group    "=== V7.33: LOSS-Based Partial Closing ==="
input bool     INPUT_ENABLE_LOSS_PARTIAL_CLOSE = true;  // Enable loss partial closing
input double   INPUT_LOSS_CLOSE_PERCENT = 50.0;          // % of lots to close when loss trigger hit
input int      INPUT_LOSS_PARTS_COUNT = 1;               // Number of closing parts (1=single, 2=two-part, 3=three-part, etc.)
input string   INPUT_LOSS_PARTS_PERCENTAGES = "50";      // Close percentages per part (comma-separated, e.g. "33,33,34" for 3 parts)
input string   INPUT_LOSS_PARTS_TRIGGERS = "50";         // Trigger percentages per part (comma-separated, e.g. "30,60,90")

input group    "=== V7.33: PROFIT-Based Partial Closing ==="
input bool     INPUT_ENABLE_PROFIT_PARTIAL_CLOSE = true; // Enable profit partial closing
input double   INPUT_PROFIT_CLOSE_PERCENT = 50.0;         // % of lots to close when profit trigger hit
input int      INPUT_PROFIT_PARTS_COUNT = 1;              // Number of closing parts (1=single, 2=two-part, 3=three-part, etc.)
input string   INPUT_PROFIT_PARTS_PERCENTAGES = "50";     // Close percentages per part (comma-separated, e.g. "33,33,34")
input string   INPUT_PROFIT_PARTS_TRIGGERS = "50";        // Trigger percentages per part (comma-separated, e.g. "30,60,90")

input group    "=== V7.33: Trailing TP Gap Feature ==="
input bool     INPUT_ENABLE_TRAILING_TP_GAP = true;       // Enable trailing TP with gap
input double   INPUT_TRAILING_TP_GAP_POINTS = 100.0;      // Gap between TP and current price (points)
input double   INPUT_TRAILING_TP_STEP_POINTS = 50.0;      // Minimum movement step (points)
input double   INPUT_TRAILING_TP_ACTIVATION_POINTS = 200.0; // Activate after this profit (points)
input group    "=== Money Management / Streak Multiplier ==="
input bool     INPUT_ENABLE_STREAK_LOT_MULTIPLIER = true; // Enable temporary lot multiplier after win streak
input int      INPUT_STREAK_TRIGGER_WINS = 2; // Consecutive wins needed to arm streak multiplier
input double   INPUT_STREAK_LOT_MULTIPLIER = 1.5; // Lot multiplier during armed streak window
input int      INPUT_STREAK_MULTIPLIER_ORDERS = 3; // Number of successful orders that use streak multiplier
input bool     INPUT_ENABLE_CONSEC_WIN_CONF_BOOST = false;
input int      INPUT_CONSEC_WIN_CONF_TRIGGER = 3;
input double   INPUT_CONSEC_WIN_CONF_BOOST_PER_WIN = 1.5;
input double   INPUT_CONSEC_WIN_CONF_BOOST_CAP = 8.0;
input bool     INPUT_ENABLE_CONSEC_WIN_CONF_DECAY = true;
input int      INPUT_CONSEC_WIN_CONF_DECAY_AFTER_TRADES = 3;
//--- Recovery Averaging System
input group    "=== Recovery Averaging System ==="
input bool     INPUT_ENABLE_RECOVERY         = false; // Master recovery gate
input ENUM_RECOVERY_MODE INPUT_RECOVERY_MODE = RECOVERY_AVERAGING; // Recovery mode selector
input int      INPUT_RECOVERY_THREAT_MIN     = 60;   // Minimum threat to trigger recovery
input int      INPUT_MAX_RECOVERY_PER_POS    = 2;    // Max recovery orders per position
input double   INPUT_RECOVERY_LOT_RATIO_SAFE = 0.33; // Lot ratio when threat < 50
input double   INPUT_RECOVERY_LOT_RATIO_MOD  = 0.50; // Lot ratio when threat 50-70
input double   INPUT_RECOVERY_LOT_RATIO_HIGH = 0.75; // Lot ratio when threat > 70
input int      INPUT_RECOVERY_TIMEOUT_MINUTES = 120; // Recovery order timeout (minutes)
input double   INPUT_RECOVERY_TRIGGER_DEPTH  = 40.0; // Trigger at X% of SL distance
input double   INPUT_RECOVERY_TP_BUFFER_POINTS = 60.0; // Add this many points beyond combined break-even for recovery TP
input double   INPUT_RECOVERY_TP_TARGET_MULTIPLIER = 1.0; // Optional target model multiplier on (combined BE-to-SL) distance
input int      INPUT_RECOVERY_COOLDOWN_SECONDS = 30; // Cooldown between recovery attempts
input int      INPUT_RECOVERY_MAX_LAYERS = 3; // Safety cap for recovery layering
input double   INPUT_RECOVERY_EMERGENCY_STOP_PERCENT = 20.0; // Halt recovery when drawdown exceeds this
input double   INPUT_GRID_STEP_POINTS = 150.0; // Grid mode spacing
input int      INPUT_GRID_MAX_ORDERS = 3; // Grid max recovery orders
input double   INPUT_GRID_LOT_SCALING = 1.0; // Grid lot scaling
input double   INPUT_HEDGE_TRIGGER_OFFSET_POINTS = 120.0; // Hedge trigger offset in points
input double   INPUT_HEDGE_LOT_SCALING = 1.0; // Hedge lot scaling
input int      INPUT_HEDGE_MAX_ORDERS = 2; // Hedge max recovery orders
input double   INPUT_MARTINGALE_MULTIPLIER = 1.6; // Martingale lot multiplier
input int      INPUT_MARTINGALE_MAX_ORDERS = 2; // Martingale max recovery orders
//--- Session Filters
input group    "=== Session Filters ==="
input bool     INPUT_TRADE_ASIAN    = true;       // Trade Asian session
input bool     INPUT_TRADE_LONDON   = true;       // Trade London session
input bool     INPUT_TRADE_NEWYORK  = true;       // Trade New York session
input int      INPUT_ASIAN_START    = 0;          // Asian start hour (server)
input int      INPUT_ASIAN_END      = 8;          // Asian end hour
input int      INPUT_LONDON_START   = 8;          // London start hour
input int      INPUT_LONDON_END     = 16;         // London end hour
input int      INPUT_NY_START       = 13;         // NY start hour
input int      INPUT_NY_END         = 22;         // NY end hour (FIXED: Extended from 21)
input int      INPUT_FRIDAY_LATE_HOUR_UTC = 18;   // Factor 7: apply Friday liquidity penalty only after this UTC hour
input double   INPUT_FRIDAY_LATE_PENALTY = 5.0;     // Factor 7: late-Friday liquidity penalty points
input bool     INPUT_ENABLE_END_OF_MONTH_PENALTY = true; // Factor 7: toggle month-end risk uplift
input int      INPUT_END_OF_MONTH_START_DAY = 29;   // Factor 7: apply month-end penalty from this day onward
input double   INPUT_END_OF_MONTH_PENALTY = 2.0;    // Factor 7: lower default month-end penalty
//--- Q-Learning System
input group    "=== Q-Learning System ==="
input bool     INPUT_ENABLE_RL          = false;  // Enable Reinforcement Learning (FIXED: Disabled for backtesting)
input double   INPUT_RL_ALPHA           = 0.1;    // Learning rate (alpha)
input double   INPUT_RL_GAMMA           = 0.9;    // Discount factor (gamma)
input double   INPUT_RL_EPSILON         = 0.1;    // Exploration rate (epsilon)
input int      INPUT_RL_MIN_TRADES      = 20;     // Min trades before RL applies
input double   INPUT_RL_WEIGHT          = 0.3;    // RL weight in final decision (0-1)
input bool     INPUT_RL_USE_RAW_REWARD  = false;  // Use raw netProfit reward instead of normalized profit/risk
input int      INPUT_RL_PENDING_HARD_CAP = 500;   // Hard cap for pending RL entries
input bool     INPUT_STRICT_STATE_LOAD  = true;   // Strict runtime load: reset pending RL/watermarks on checksum mismatch
//--- Markov Chain Analysis
input group    "=== Markov Chain Analysis ==="
input bool     INPUT_ENABLE_MARKOV      = false;  // Enable Markov chain analysis (FIXED: Disabled for backtesting)
input int      INPUT_MARKOV_LOOKBACK    = 100;    // Lookback for transition matrix
input double   INPUT_STREAK_FATIGUE_ADJ = 0.05;   // Confidence reduction per streak trade
input double   INPUT_MARKOV_WIN_R       = 0.1;    // Win threshold in normalized R units
input double   INPUT_MARKOV_LOSS_R      = -0.1;   // Loss threshold in normalized R units
//--- Machine Learning
input group    "=== Machine Learning ==="
input bool     INPUT_ENABLE_ML          = false;  // Enable ML signal analysis (FIXED: Disabled for backtesting)
input bool     INPUT_ENABLE_FINGERPRINT = false;  // Enable fingerprint learning (FIXED: Disabled for backtesting)
input int      INPUT_MIN_TRADES_FOR_ML  = 10;     // Min trades before ML applies
input double   INPUT_LEARNING_DECAY     = 0.98;   // Learning decay factor
input int      INPUT_MAX_TRAINING_DATA  = 1000;   // Max training records
input bool     INPUT_RESET_LEGACY_SESSION_DATA = false; // Reset legacy datasets saved with wrong session/day attribution
//--- DeepSeek AI Integration
input group    "=== DeepSeek AI Integration ==="
input ENUM_AI_MODE INPUT_AI_MODE = AI_OFF;        // AI Mode
input string   INPUT_AI_API_KEY  = "";            // DeepSeek API Key (sk-...)
input string   INPUT_AI_URL      = "https://api.deepseek.com/v1/chat/completions";
input int      INPUT_AI_INTERVAL_MINUTES = 15;    // API query interval (minutes)
input double   INPUT_AI_WEIGHT   = 0.2;           // AI weight in confidence (0-1)
//--- Adaptive Parameters
input group    "=== Adaptive Parameters ==="
input bool     INPUT_ENABLE_ADAPTIVE    = false;  // Enable adaptive optimization (FIXED: Disabled for backtesting)
input int      INPUT_ADAPT_INTERVAL     = 50;     // Optimize every N trades
input double   INPUT_ADAPT_UNDERPERF_LOT_REDUCE = 0.1; // Reduce lots by X when underperforming
input double   INPUT_ADAPT_OVERPERF_TRAIL_ADD   = 2.0; // Add X pips to trail when outperforming
input bool     INPUT_ENABLE_COMBINATION_ADAPTIVE = true; // Priority 2: learn per-signal-combination behavior
input bool     INPUT_ENABLE_FULL_COMBO_UNIVERSE = true; // Pre-seed deterministic nCk universe
input int      INPUT_TOTAL_SIGNALS = 8;           // Total available signal factors (n)
input int      INPUT_TOTAL_SIGNAL_FACTORS = 8;    // Alias for total factors (n)
input int      INPUT_COMBO_MIN_TRADES   = 10;     // Minimum trades required per combination for analysis
input double   INPUT_COMBO_CONFIDENCE_WEIGHT = 0.3; // How strongly combo strength affects confidence
input ENUM_COMBO_RANK_MODE INPUT_COMBO_RANK_MODE = COMBO_RANK_HEURISTIC;
input bool     INPUT_LOG_COMBINATION_INSIGHTS = true; // Print best/worst combination insights
input int      INPUT_COMBO_INSIGHT_TOP_N = 1;     // Number of best/worst combos to log per refresh
input bool     INPUT_ENABLE_TREE_FEATURE_MODULE = false; // Decision-tree subset feature ranking
input bool     INPUT_TREE_ADJUST_CONFIDENCE_ON = false; // Apply selected tree features to confidence
input bool     INPUT_TREE_ENTRY_GATE_ON = false; // Require minimum selected-feature matches
input int      INPUT_TREE_BRANCH_MIN_SUPPORT = 10; // Minimum branch support for feature IG
input int      INPUT_TREE_MAX_SELECTED_FEATURES = 5; // Max features selected by greedy IG
input int      INPUT_TREE_MIN_SELECTED_MATCH = 1; // Minimum selected-feature matches for entry gate
input double   INPUT_TREE_MIN_IG = 0.0001; // Minimum IG to keep feature
input double   INPUT_TREE_CONFIDENCE_WEIGHT = 0.15; // Confidence adjustment weight from selected features
input bool     INPUT_AGE_TIMEOUT_INCLUDE_AUX = false; // Include recovery/aux positions in age-timeout close
input int      INPUT_POSITION_AGE_CHECK_SECONDS = 5; // Throttle stale-position timeout checks
input int      INPUT_HISTORY_PROCESS_INTERVAL_SECONDS = 2; // Throttle closed-deal history scan
input int      INPUT_HISTORY_BOOTSTRAP_DAYS = 7; // Initial history window (days) when no watermark exists
input int      INPUT_HISTORY_SAFETY_MARGIN_SECONDS = 300; // History overlap to avoid missing boundary deals
input int      INPUT_STATE_CHECKPOINT_MINUTES = 5; // Periodic persistence checkpoint
input int      INPUT_RL_PENDING_MAX_AGE_HOURS = 72; // Expire unmatched RL pending entries
input int      INPUT_ON_TICK_BUDGET_MS = 30;      // Soft per-tick budget for non-critical work
input double   INPUT_MIN_EFFECTIVE_RR_AFTER_SPREAD = 1.05; // Minimum net RR after spread for new entries
input int      INPUT_SERVER_UTC_OFFSET_HOURS = 0; // Broker server UTC offset for DST-aware session mapping
input bool     INPUT_ENABLE_META_POLICY = true; // Blend rule policy + RL using state confidence
input int      INPUT_RL_MIN_STATE_VISITS = 8; // Minimum state visits before RL can override baseline
input bool     INPUT_ENABLE_RISK_PARITY_CAP = true; // Volatility/session normalized lot cap
input double   INPUT_RISK_PARITY_BASE_CAP_LOTS = 1.0; // Base lot cap for risk-parity normalizer
input int      INPUT_DATA_WARNING_KILL_SWITCH = 25; // Halt entries when rolling integrity warnings exceed threshold
input int      INPUT_DATA_WARNING_WINDOW_MINUTES = 30; // Rolling window length for anomaly kill-switch
input int      INPUT_HEAVY_BASE_INTERVAL_SECONDS = 2; // Base throttle for expensive maintenance tasks
//--- Indicator Settings
input group    "=== Indicator Settings ==="
input int      INPUT_EMA_FAST        = 8;         // Fast EMA period
input int      INPUT_EMA_SLOW        = 21;        // Slow EMA period
input int      INPUT_EMA_TREND       = 200;       // Trend EMA period
input int      INPUT_RSI_PERIOD      = 14;        // RSI period
input int      INPUT_STOCH_K         = 14;        // Stochastic %K
input int      INPUT_STOCH_D         = 3;         // Stochastic %D
input int      INPUT_STOCH_SLOW      = 3;         // Stochastic slowing
input int      INPUT_MACD_FAST       = 12;        // MACD fast
input int      INPUT_MACD_SLOW       = 26;        // MACD slow
input int      INPUT_MACD_SIGNAL     = 9;         // MACD signal
input int      INPUT_WPR_PERIOD      = 14;        // Williams %R period
input int      INPUT_ATR_PERIOD      = 14;        // ATR period
input double   INPUT_EMA_SLOPE_ATR_WEAK = 0.005; // EMA slope weak threshold as ATR fraction per bar (symbol-agnostic)
input double   INPUT_EMA_SLOPE_ATR_STRONG = 0.010; // EMA slope strong threshold as ATR fraction per bar (symbol-agnostic)
input int      INPUT_ADX_PERIOD      = 14;        // ADX period
input int      INPUT_BB_PERIOD       = 20;        // Bollinger Bands period
input double   INPUT_BB_DEVIATION    = 2.0;       // BB deviation
input int      INPUT_BREAKOUT_LOOKBACK = 20;      // Breakout lookback bars
input int      INPUT_VOLUME_AVG_PERIOD = 20;      // Volume average period
//--- Debug & Display
input group    "=== Debug & Display ==="
input bool     INPUT_ENABLE_LOGGING   = true;     // Enable detailed logging
input bool     INPUT_SHOW_PANEL       = true;     // Show on-chart panel
input bool     INPUT_ENABLE_ALERTS    = false;    // Enable alert notifications
input int      INPUT_REPEAT_LOG_RESTART_THRESHOLD = 50; // Restart EA when exact same log repeats this many times
input int      INPUT_CLOSED_DEALS_MAX_ROWS = 5000; // Max rows to keep in closed deals csv (0=unlimited)
//+------------------------------------------------------------------+
//| SECTION 3: CONSTANTS                                             |
//+------------------------------------------------------------------+
#define COMMENT_MAIN_PREFIX       "V7_MAIN_"
#define COMMENT_RECOVERY_PREFIX   "V7_REC_"
#define COMMENT_AVG_PREFIX        "V7_AVG_"
#define COMMENT_HEDGE_PREFIX      "V7_HDG_"
#define COMMENT_GRID_PREFIX       "V7_GRD_"
#define COMMENT_50PCT_PREFIX      "V7_50P_"
#define MAX_POSITIONS             100
#define MAX_FINGERPRINTS          500
#define MAX_TRAINING_DATA         1000
#define MAX_COMBINATION_STATS     200
#define Q_TABLE_STATES            108
#define Q_TABLE_ACTIONS           4
#define MARKOV_STATES             3
#define QTABLE_SCHEMA_VERSION     3
#define RUNTIME_SCHEMA_VERSION    8
#define EA_VERSION_LABEL          "V7.3"
#define QTABLE_HASH_SENTINEL      0x51424C31
#define RUNTIME_HASH_SENTINEL     0x52554E31

enum ENUM_POSITION_SUBTYPE
{
   SUBTYPE_MAIN      = 0,
   SUBTYPE_RECOVERY  = 1,
   SUBTYPE_AVERAGING = 2,
   SUBTYPE_AUX       = 3
};

const long MAGIC_SUBTYPE_MULTIPLIER = 100000000;
const long MAGIC_BASE_MIN = 1;
const long MAGIC_BASE_MAX = MAGIC_SUBTYPE_MULTIPLIER - 1;
const int  AI_TIMER_SECONDS = 1;
//+------------------------------------------------------------------+
//| SECTION 4: STRUCTURES                                            |
//+------------------------------------------------------------------+
struct RiskParams
{
   double riskPercent;
   double maxLot;
   double minLot;
   double maxTotalRisk;
};
struct AdaptiveParams
{
   double lotMultiplier;        // Dynamic lot multiplier
   double slAdjustPoints;         // SL adjustment in points
   double tpAdjustPoints;         // TP adjustment in points
   double trailAdjustPoints;      // Trail adjustment in points
   double threatMultiplier;     // Threat calculation multiplier
   double confMultiplierCap;    // Max ML confidence multiplier
   double minConfThreshold;     // Dynamic min confidence
   int    maxPositions;         // Dynamic max positions
   datetime lastOptimization;   // Last optimization time
   int    tradesAtLastOpt;      // Trades count at last optimization
};
struct PositionState
{
   ulong    ticket;
   int      direction;          // 1=BUY, -1=SELL
   double   entryPrice;
   double   slPrice;
   double   tpPrice;
   double   originalLots;
   double   currentLots;
   string   signalCombination;
   string   comment;
   datetime entryTime;
   int      entrySession;
   int      entryDayOfWeek;
   ENUM_MARKET_REGIME entryRegime;
   double   confidenceAtEntry;
   double   threatAtEntry;
   int      mtfScoreAtEntry;
   string   fingerprintId;
   bool     halfSLHit;          // 50% SL distance reached
   bool     lotReduced;         // 50% lot already closed
   bool     partialClosed;      // Partial close at TP% done
    bool     multiPartialLevel1Done; // 30% progress partial done
   bool     multiPartialLevel2Done; // 60% progress partial done
   bool     movedToBreakeven;   // SL moved to breakeven
   int      recoveryCount;      // How many recovery orders placed
   datetime lastRecoveryTime;   // Last recovery order time
   bool     isActive;           // Position still open
   double   maxProfit;          // Max profit seen (for trailing)
   double   maxLoss;
   
   // V7.33 NEW FIELDS
   bool     lossPartialLevel1Done;    // First loss partial close done
   bool     lossPartialLevel2Done;    // Second loss partial close done
   bool     lossPartialLevel3Done;    // Third loss partial close done
   bool     lossPartialLevel4Done;    // Fourth loss partial close done
   bool     profitPartialLevel1Done;  // First profit partial close done
   bool     profitPartialLevel2Done;  // Second profit partial close done
   bool     profitPartialLevel3Done;  // Third profit partial close done
   bool     profitPartialLevel4Done;  // Fourth profit partial close done
   bool     trailingTPActive;         // Trailing TP activated
   double   lastTrailingTPPrice;      // Last trailing TP price            // Max loss seen (for analysis)
};
struct SignalResult
{
   bool     emaSignal;
   bool     rsiSignal;
   bool     stochSignal;
   bool     engulfingSignal;
   bool     breakoutSignal;
   bool     volumeSignal;
   bool     macdSignal;
   bool     wprSignal;
   int      bullVotes;
   int      bearVotes;
   int      totalSignals;
   string   combinationString;
};
struct DecisionResult
{
   bool     shouldTrade;
   int      direction;          // 1=BUY, -1=SELL
   double   confidence;
   double   threatLevel;
   ENUM_THREAT_ZONE threatZone;
   int      mtfScore;
   int      signalCount;
   string   signalCombination;
   double   slPoints;
   double   tpPoints;
   double   lotSize;
   string   fingerprintId;
   ENUM_RL_ACTION rlAction;
};
struct SignalFingerprint
{
   string   id;
   string   signalCombination;
   int      session;            // 0=Asian, 1=London, 2=NY
   int      dayOfWeek;
   ENUM_MARKET_REGIME regime;
   int      totalOccurrences;
   int      wins;
   int      losses;
   double   totalProfit;
   double   totalLoss;
   double   winRate;
   double   profitFactor;
   double   avgProfit;
   double   avgLoss;
   double   expectancy;
   double   entropy;
   double   infoGain;
   double   rankScore;
   double   strengthScore;      // 0-100
   double   confidenceMultiplier; // 0.5-1.5
   double   decayWeight;
   datetime lastSeen;
};
struct TrainingData
{
   ulong    ticket;
   datetime entryTime;
   datetime closeTime;
   int      holdingMinutes;
   string   signalCombination;
   double   entryPrice;
   double   slPrice;
   double   tpPrice;
   double   closePrice;
   string   exitType;           // "TP", "SL", "Trailing", "Manual", "50pct", "Recovery"
   double   profitLoss;
   bool     isWin;
   double   confidenceAtEntry;
   double   threatAtEntry;
   int      mtfScore;
   double   volatilityRatio;
   int      entrySession;
   int      closeSession;
   int      entryDayOfWeek;
   int      closeDayOfWeek;
   ENUM_MARKET_REGIME entryRegime;
   ENUM_MARKET_REGIME closeRegime;
   string   fingerprintId;
   int      rlState;
   ENUM_RL_ACTION rlAction;
};
struct CombinationStats
{
   string   combination;
   string   comboId;
   bool     seen;
   int      totalTrades;
   int      wins;
   int      losses;
   double   totalProfit;
   double   totalLoss;
   double   winRate;
   double   profitFactor;
   double   avgProfit;
   double   avgLoss;
   double   expectancy;
   double   entropy;
   double   infoGain;
   double   rankScore;
   double   strengthScore;      // 0-100
   double   confidenceMultiplier; // 0.5-1.5
   // Session breakdown
   int      asianWins;
   int      asianTotal;
   int      londonWins;
   int      londonTotal;
   int      nyWins;
   int      nyTotal;
   // Regime breakdown
   int      trendingWins;
   int      trendingTotal;
   int      rangingWins;
   int      rangingTotal;
};
struct TreeFeatureMetric
{
   string   feature;
   int      support;
   int      yesWins;
   int      yesLosses;
   int      noWins;
   int      noLosses;
   double   entropyYes;
   double   entropyNo;
   double   infoGain;
   bool     selected;
};
struct DailyStats
{
   datetime dayStart;
   double   dayStartBalance;
   int      tradesPlaced;
   int      pendingOrdersPlaced;
   int      closedDealsToday;
   int      winsToday;
   int      lossesToday;
   double   profitToday;
   double   lossToday;
   double   peakEquityToday;
   double   realizedDealPnlToday;        // Deal-leg cashflow (can include partial closes)
   double   realizedFinalPositionPnlToday; // Terminal position outcome PnL
   int      strategyWinsToday;           // Strategy-level terminal wins
   int      strategyLossesToday;         // Strategy-level terminal losses
};
struct GateDiagnostics
{
   int      sessionRejects;
   int      cooldownRejects;
   int      signalsRejects;
   int      mtfRejects;
      int      mtfDataReadRejects;
   int      adxDataReadRejects;
   int      threatRejects;
   int      confidenceRejects;
   int      maxPositionsRejects;
};
struct RLStateAction
{
   int      state;
   ENUM_RL_ACTION action;
   datetime timestamp;
   ulong    orderTicket;
   ulong    positionTicket;
   double   entryPrice;
   double   slDistance;
   double   lot;
   double   tickValue;
   double   confidenceSnapshot;
   int      mtfScoreSnapshot;
   double   comboStrengthSnapshot;
};
struct ClosedPositionContext
{
   PositionState state;
   datetime archivedAt;
};
struct MarkovTransitionEvent
{
   ENUM_MARKOV_STATE fromState;
   ENUM_MARKOV_STATE toState;
   datetime          observedAt;
};
struct AIResponse
{
   string   marketBias;         // bullish/bearish/neutral
   double   confidenceScore;    // 0-100
   bool     riskAlert;
   datetime lastUpdate;
   int      consecutiveErrors;
};
struct PositionCloseAccumulator
{
   ulong    positionId;
   double   cumulativeNetProfit;
   double   closedVolume;
   double   openedVolume;
   datetime firstCloseTime;
   datetime lastCloseTime;
   ulong    lastDealTicket;
   bool     terminalByHistory;
};
//+------------------------------------------------------------------+
//| SECTION 5: GLOBAL VARIABLES                                      |
//+------------------------------------------------------------------+
//--- Trade object
CTrade g_trade;
CPositionInfo g_posInfo;
//--- Symbol specs
double   g_point;
double   g_tickSize;
double   g_tickValue;
double   g_lotStep;
int      g_lotDigits;
double   g_minLot;

// V7.33: Arrays for multi-part closing
double   g_lossPartPercentages[10];   // Array to store parsed loss close percentages
double   g_lossPartTriggers[10];      // Array to store parsed loss trigger percentages
double   g_profitPartPercentages[10]; // Array to store parsed profit close percentages
double   g_profitPartTriggers[10];    // Array to store parsed profit trigger percentages
int      g_lossPartsCount = 0;        // Actual count of loss parts
int      g_profitPartsCount = 0;      // Actual count of profit parts

double   g_maxLot;
int      g_digits;
double   g_contractSize;
long     g_stopLevel;
long     g_freezeLevel;
int      g_dataIntegrityWarnings = 0;
datetime g_dataWarningWindowStart = 0;
bool     g_lotStepFallbackLogged = false;
bool     g_mtfReadFailureThisTick = false;
bool     g_lastMtfAlignmentHadReadFailure = false;
bool     g_lastMtfConsensusHadReadFailure = false;
ENUM_ORDER_TYPE_FILLING g_selectedFillingMode = ORDER_FILLING_IOC;

//--- Persistence / validation schema
#define FP_SCHEMA_VERSION         2
#define TRAINING_SCHEMA_VERSION   3
#define ADAPTIVE_SCHEMA_VERSION   2

double GetPipSize(const string symbol)
{
   int digits = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
   if(digits == 3 || digits == 5)
      return SymbolInfoDouble(symbol, SYMBOL_POINT) * 10.0;
   return SymbolInfoDouble(symbol, SYMBOL_POINT);
}

double PipsToPoints(const string symbol, double pips)
{
   double point = SymbolInfoDouble(symbol, SYMBOL_POINT);
   double pipSize = GetPipSize(symbol);
   if(point <= 0.0 || pipSize <= 0.0 || !MathIsValidNumber(pips))
      return 0.0;
   return pips * (pipSize / point);
}

void RegisterDataWarning(const string context)
{
   datetime now = TimeCurrent();
   int windowSec = MathMax(60, INPUT_DATA_WARNING_WINDOW_MINUTES * 60);
   if(g_dataWarningWindowStart == 0 || (now - g_dataWarningWindowStart) > windowSec)
   {
      g_dataWarningWindowStart = now;
      g_dataIntegrityWarnings = 0;
   }

   g_dataIntegrityWarnings++;
   Print("DATA WARNING: ", context, " | rollingWarnings=", g_dataIntegrityWarnings);
}

void RefreshSymbolTradeConstraints()
{
   long stopNow = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
   long freezeNow = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_FREEZE_LEVEL);
   if(stopNow < 0) stopNow = 0;
   if(freezeNow < 0) freezeNow = 0;

   if(stopNow != g_stopLevel || freezeNow != g_freezeLevel)
   {
      if(INPUT_ENABLE_LOGGING)
         Print("SYMBOL CONSTRAINT CHANGE: stopLevel ", g_stopLevel, "->", stopNow,
               " | freezeLevel ", g_freezeLevel, "->", freezeNow);
      g_stopLevel = stopNow;
      g_freezeLevel = freezeNow;
   }
}

bool IsFiniteInRange(double value, double minValue, double maxValue)
{
   return (MathIsValidNumber(value) && value >= minValue && value <= maxValue);
}

double GetEffectiveLotStep()
{
   if(MathIsValidNumber(g_lotStep) && g_lotStep > 0.0)
      return g_lotStep;

   double fallbackStep = (MathIsValidNumber(g_minLot) && g_minLot > 0.0) ? g_minLot : 0.01;
   if(!g_lotStepFallbackLogged)
   {
      g_lotStepFallbackLogged = true;
      LogWithRestartGuard("LOT STEP FALLBACK: invalid SYMBOL_VOLUME_STEP=" + DoubleToString(g_lotStep, 8) +
                          " using fallback=" + DoubleToString(fallbackStep, 8));
   }
   return fallbackStep;
}

double NormalizeVolumeToStep(double lots)
{
   double step = GetEffectiveLotStep();
   if(!MathIsValidNumber(lots) || lots <= 0.0 || !MathIsValidNumber(step) || step <= 0.0)
      return 0.0;

   double aligned = MathFloor((lots + 1e-12) / step) * step;
   return NormalizeDouble(aligned, g_lotDigits);
}

bool NormalizeAndValidateOrderVolume(double requestedLots, double &normalizedLots, string &reason)
{
   reason = "";
   normalizedLots = NormalizeVolumeToStep(requestedLots);
   if(normalizedLots <= 0.0)
   {
      reason = "non-positive or non-finite normalized lot";
      return false;
   }

   normalizedLots = MathMax(normalizedLots, g_risk.minLot);
   normalizedLots = MathMin(normalizedLots, g_risk.maxLot);
   normalizedLots = MathMax(normalizedLots, g_minLot);
   normalizedLots = MathMin(normalizedLots, g_maxLot);
   normalizedLots = NormalizeVolumeToStep(normalizedLots);

   if(!MathIsValidNumber(normalizedLots) || normalizedLots < g_minLot || normalizedLots > g_maxLot)
   {
      reason = "outside broker min/max volume bounds";
      return false;
   }

   double volumeLimit = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_LIMIT);
   if(MathIsValidNumber(volumeLimit) && volumeLimit > 0.0 && normalizedLots > volumeLimit)
   {
      double capped = NormalizeVolumeToStep(volumeLimit);
      if(capped < g_minLot)
      {
         reason = "symbol volume limit below minimum tradable lot";
         return false;
      }
      normalizedLots = capped;
   }

   return true;
}


void ResetAdaptiveParamsToDefaults();
bool IsPlacementEnabled() { return INPUT_TOGGLE_PLACE_ORDERS; }
bool IsCloseEnabled() { return INPUT_TOGGLE_CLOSE_ORDERS; }
bool IsStopModifyEnabled() { return INPUT_TOGGLE_MODIFY_STOPS; }
bool IsTpModifyEnabled() { return INPUT_TOGGLE_MODIFY_TPS; }

bool IsFeatureEnabled(string featureId)
{
   if(featureId == "market_orders") return (IsPlacementEnabled() && INPUT_TOGGLE_MARKET_ORDERS && INPUT_EXEC_MARKET_PATH_ON);
   if(featureId == "pending_orders") return (IsPlacementEnabled() && INPUT_TOGGLE_PENDING_ORDERS && INPUT_EXEC_PENDING_PATH_ON);
   if(featureId == "close") return IsCloseEnabled();
   if(featureId == "modify_sl") return IsStopModifyEnabled();
   if(featureId == "modify_tp") return IsTpModifyEnabled();
   return true;
}

bool ReplaceFileAtomic(const string tmpName, const string finalName)
{
   if(!FileIsExist(tmpName))
   {
      Print("ERROR: ReplaceFileAtomic missing temp file: ", tmpName, " -> ", finalName);
      return false;
   }

   int testHandle = FileOpen(tmpName, FILE_READ | FILE_BIN);
   if(testHandle == INVALID_HANDLE)
      testHandle = FileOpen(tmpName, FILE_READ | FILE_CSV | FILE_ANSI, ',');
   if(testHandle == INVALID_HANDLE)
   {
      Print("ERROR: ReplaceFileAtomic temp file unreadable: ", tmpName, " -> ", finalName);
      return false;
   }
   FileClose(testHandle);

   string backupName = finalName + ".bak";
   if(FileIsExist(backupName) && !FileDelete(backupName))
   {
      Print("ERROR: ReplaceFileAtomic failed deleting stale backup: ", backupName);
      return false;
   }

   bool hadDestination = FileIsExist(finalName);
   if(hadDestination)
   {
      if(!FileMove(finalName, 0, backupName, 0))
      {
         Print("ERROR: ReplaceFileAtomic failed moving destination to backup: ", finalName, " -> ", backupName);
         return false;
      }
   }

   if(!FileMove(tmpName, 0, finalName, 0))
   {
      Print("ERROR: ReplaceFileAtomic failed moving temp to destination: ", tmpName, " -> ", finalName);
      if(hadDestination)
      {
         if(!FileMove(backupName, 0, finalName, 0))
            Print("WARNING: ReplaceFileAtomic rollback failed: ", backupName, " -> ", finalName);
         else
            Print("WARNING: ReplaceFileAtomic rollback restored destination: ", finalName);
      }
      return false;
   }

   if(hadDestination && FileIsExist(backupName) && !FileDelete(backupName))
      Print("WARNING: ReplaceFileAtomic success but could not delete backup: ", backupName);

   return true;
}

void ResetRuntimeLinkedInMemoryState();

bool ResetAllPersistedStateFiles();

//--- Risk
RiskParams g_risk;
AdaptiveParams g_adaptive;
//--- Trading state
ENUM_EA_STATE g_eaState = STATE_IDLE;
ENUM_EA_STATE g_prevEaState = STATE_IDLE;
PositionState g_positions[];
int      g_positionCount = 0;
datetime g_lastOrderTime = 0;
datetime g_lastBuyOrderTime = 0;
datetime g_lastSellOrderTime = 0;
bool     g_currentEntryIsFlip = false;
bool     g_flipCleanupInProgress = false;
bool     g_flipCooldownBypassLogged = false;
datetime g_lastBarTime = 0;
double   g_peakEquity = 0;
double   g_startingBalance = 0;
int      g_consecutiveLosses = 0;
int      g_consecutiveWins = 0;
int      g_streakMultiplierOrdersRemaining = 0;
ulong    g_lastStreakActivatedDeal = 0;
double   g_averageSpread = 0;
int      g_spreadSamples = 0;
double   g_totalSpread = 0;
double   g_averageATR = 0;
int      g_totalTrades = 0;
//--- Daily tracking
DailyStats g_daily;
GateDiagnostics g_gateDiagnostics;
//--- Market regime
ENUM_MARKET_REGIME g_currentRegime = REGIME_UNKNOWN;
//--- Indicator handles - M1
int      g_hEmaFast_M1, g_hEmaSlow_M1, g_hEmaTrend_M1;
int      g_hRSI_M1;
int      g_hStoch_M1;
int      g_hMACD_M1;
int      g_hWPR_M1;
int      g_hATR_M1;
int      g_hADX_M1;
int      g_hBB_M1;
int      g_hVolume_M1;
//--- Indicator handles - M5
int      g_hEmaFast_M5, g_hEmaSlow_M5;
int      g_hATR_M5;
//--- Indicator handles - H1
int      g_hEmaFast_H1, g_hEmaSlow_H1;
int      g_hATR_H1;
int      g_hADX_H1;
//--- Indicator handles - H4
int      g_hEmaFast_H4, g_hEmaSlow_H4;
//--- Indicator handles - D1
int      g_hEmaFast_D1, g_hEmaSlow_D1;
//--- Learning system
SignalFingerprint g_fingerprints[];
int      g_fingerprintCount = 0;
//--- ML Training
TrainingData g_trainingData[];
int      g_trainingDataCount = 0;
CombinationStats g_combinationStats[];
int      g_combinationStatsCount = 0;
string   g_comboUniverse[];
int      g_comboUniverseCount = 0;
int      g_comboObservedCount = 0;
TreeFeatureMetric g_treeFeatureMetrics[];
int      g_treeFeatureMetricCount = 0;
string   g_treeSelectedFeatures[];
int      g_treeSelectedFeatureCount = 0;
double   g_treeParentEntropy = 0.0;
int      g_consecWinBoostTrades = 0;
string   g_activeRecoveryPrefix = COMMENT_AVG_PREFIX;
ENUM_POSITION_SUBTYPE g_activeRecoverySubtype = SUBTYPE_AVERAGING;
//--- Q-Learning System (108 states x 4 actions)
double   g_qTable[Q_TABLE_STATES][Q_TABLE_ACTIONS];
int      g_qVisits[Q_TABLE_STATES][Q_TABLE_ACTIONS];
RLStateAction g_pendingRL[];
int      g_pendingRLCount = 0;
int      g_rlTradesCompleted = 0;
ClosedPositionContext g_recentlyClosedContext[];
int      g_recentlyClosedContextCount = 0;
datetime g_lastExtremeResolvedLogTime = 0;
string   g_logSuppressionKeys[];
datetime g_logSuppressionTimes[];
int      g_logSuppressionCount = 0;
//--- Markov Chain (3x3 transition matrix)
double   g_markovTransitions[MARKOV_STATES][MARKOV_STATES];
int      g_markovCounts[MARKOV_STATES][MARKOV_STATES];
ENUM_MARKOV_STATE g_lastMarkovState = MARKOV_EVEN;
int      g_markovTradesRecorded = 0;
MarkovTransitionEvent g_markovQueue[];
int      g_markovQueueCount = 0;
int      g_markovQueueHead = 0;

ulong    g_tickMsHistory = 0;
ulong    g_tickMsManagePositions = 0;
ulong    g_tickMsDecision = 0;
ulong    g_tickMsPanel = 0;
ulong    g_tickMsPersistence = 0;
datetime g_lastHeavyHistoryRun = 0;
datetime g_lastPanelRun = 0;
ulong    g_tickMsAIRequest = 0;
ulong    g_tickMsAILastDuration = 0;
//--- AI Integration
AIResponse g_aiResponse;
datetime g_lastAIQuery = 0;
AIResponse g_lastValidAIResponse;
datetime g_aiBackoffUntil = 0;
int      g_aiConsecutiveTransportFailures = 0;
int      g_aiSkippedByBackoff = 0;
bool     g_rngSeeded = false;
PositionCloseAccumulator g_positionCloseAccumulators[];
int      g_positionCloseAccumulatorCount = 0;
//--- History tracking
ulong    g_lastProcessedDealTicket = 0;
datetime g_lastProcessedDealTime = 0;
ulong    g_lastProcessedEntryDealTicket = 0;
datetime g_lastProcessedEntryDealTime = 0;
datetime g_lastPositionAgeCheck = 0;
datetime g_lastTrailingTPCheck = 0;
datetime g_lastHistoryProcessTime = 0;
datetime g_lastCheckpointTime = 0;
int      g_closedDealsProcessedTotal = 0;
int      g_rlMatchedUpdates = 0;
int      g_rlUnmatchedCloses = 0;
int      g_rlRuntimeRejectBadStateAction = 0;
int      g_rlRuntimeRejectTicketMismatch = 0;
int      g_rlRuntimeRejectNaNRiskBasis = 0;
int      g_syncMissingCount = 0;
int      g_syncNewCount = 0;
int      g_syncDuplicateCount = 0;
string   g_lastLogMessage = "";
int      g_lastLogRepeatCount = 0;
struct ModifyFailureTracker
{
   ulong    ticket;
   int      failCount;
   datetime nextRetryTime;
};
ModifyFailureTracker g_tpModifyFailures[];

// EXTREME_RISK_TOGGLE_COVERAGE_CHECKLIST
// - UpdateEAState
// - OnTick extreme branch
// - HandleExtremeRisk
// - Equity floor block
// - CloseAllPositions
// - CheckAllGates protection-state gate
// - RunDecisionPipeline threat blocks
// - CheckRecoveryTimeouts
// - CheckPositionAgeTimeout
// - HandleHighSpreadOpenPositions
// - Handle50PercentLotClose
// - ManagePartialClose
// - HandleMultiLevelPartial
// - ManageTrailingStops
// - ManageTrailingTP
// - MoveToBreakeven
// - ShouldSkipStopAdjustmentsForTicket

bool g_effExtremeByThreat = false;
bool g_effExtremeByDrawdown = false;
bool g_effExtremeHysteresisExit = false;
bool g_effDrawdownProtectState = false;
bool g_effExtremeOnTickHandler = false;
bool g_effExtremeOnTickEarlyReturn = false;
bool g_effExtremeCloseOldest = false;
bool g_effExtremeFilterSymbol = false;
bool g_effExtremeFilterMagic = false;
bool g_effExtremeThrottle = false;
bool g_effEquityFloorTrigger = false;
bool g_effEquityFloorForceState = false;
bool g_effEquityFloorCloseAll = false;
bool g_effEquityFloorReturn = false;
bool g_effCloseAllApi = false;
bool g_effCloseAllOnlyOur = false;
bool g_effCloseAllSymbolFilter = false;
bool g_effGateProtectionBlock = false;
bool g_effThreatHardBlock = false;
bool g_effThreatExtremeZoneBlock = false;
bool g_effThreatSoftLotShrink = false;
bool g_effCloseRecoveryTimeout = false;
bool g_effClosePositionAgeTimeout = false;
bool g_effCloseHighSpreadProfit = false;
bool g_effClose50PctDefensive = false;
bool g_effClosePartialTP = false;
bool g_effCloseMultiLevelPartial = false;
bool g_effModifyMoveToBE = false;
bool g_effModifyTrailingSL = false;
bool g_effModifyTrailingTP = false;
bool g_effModifySkipLossOnHighSpread = false;

struct EffectiveConfig
{
   bool entry;
   bool close;
   bool modifySL;
   bool modifyTP;
   bool extremeRisk;
   bool markovInfer;
   bool markovUpdate;
   bool rlRecord;
   bool rlLearn;
   bool rlInfer;
   bool mlRecord;
   bool mlInfer;
};

EffectiveConfig g_effectiveConfig;

void ResetAdaptiveParamsToDefaults()
{
   g_adaptive.lotMultiplier = 1.0;
   g_adaptive.slAdjustPoints = 0;
   g_adaptive.tpAdjustPoints = 0;
   g_adaptive.trailAdjustPoints = 0;
   g_adaptive.threatMultiplier = 1.0;
   g_adaptive.confMultiplierCap = 1.5;
   g_adaptive.minConfThreshold = INPUT_MIN_CONFIDENCE;
   g_adaptive.maxPositions = INPUT_MAX_CONCURRENT_TRADES;
   g_adaptive.lastOptimization = 0;
   g_adaptive.tradesAtLastOpt = 0;
}

bool ResolveRuntimeToggle(bool legacyFlag, bool newToggle)
{
   if(INPUT_TOGGLE_RESOLUTION_MODE == TOGGLE_RESOLUTION_MIGRATION)
   {
      if(INPUT_USE_LEGACY_BEHAVIOR_MAPPING && legacyFlag && !newToggle)
         Print("MIGRATION TOGGLE OVERRIDE: legacy=ON forces effective ON while new=OFF.");
      return (INPUT_USE_LEGACY_BEHAVIOR_MAPPING ? (legacyFlag || newToggle) : newToggle);
   }

   if(INPUT_TOGGLE_RESOLUTION_MODE == TOGGLE_RESOLUTION_STRICT_NEW && legacyFlag != newToggle)
      Print("STRICT TOGGLE MISMATCH: legacy=", (legacyFlag ? "ON" : "OFF"), " new=", (newToggle ? "ON" : "OFF"), " effective uses NEW toggle.");

   if(INPUT_FORCE_NEW_TOGGLES_ONLY)
      return newToggle;
   return newToggle;
}

void BuildEffectiveConfig()
{
   g_effectiveConfig.entry = INPUT_TOGGLE_PLACE_ORDERS && (INPUT_TOGGLE_MARKET_ORDERS || INPUT_TOGGLE_PENDING_ORDERS);
   g_effectiveConfig.close = INPUT_TOGGLE_CLOSE_ORDERS;
   g_effectiveConfig.modifySL = INPUT_TOGGLE_MODIFY_STOPS;
   g_effectiveConfig.modifyTP = INPUT_TOGGLE_MODIFY_TPS;
   g_effectiveConfig.extremeRisk = (g_effExtremeByThreat || g_effExtremeByDrawdown || g_effDrawdownProtectState || g_effExtremeCloseOldest);
   g_effectiveConfig.markovInfer = (INPUT_ENABLE_MARKOV && INPUT_MARKOV_INFERENCE_ON);
   g_effectiveConfig.markovUpdate = (INPUT_ENABLE_MARKOV && INPUT_MARKOV_UPDATE_ON);
   g_effectiveConfig.rlRecord = (INPUT_ENABLE_RL && INPUT_EXEC_RECORD_RL_ON_SUBMIT);
   g_effectiveConfig.rlLearn = (INPUT_ENABLE_RL && INPUT_RL_LEARNING_ON);
   g_effectiveConfig.rlInfer = (INPUT_ENABLE_RL && INPUT_RL_INFERENCE_ON);
   g_effectiveConfig.mlRecord = ((INPUT_ENABLE_ML && INPUT_ML_RECORD_ON) ||
                                (INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_RECORD_ON));
   g_effectiveConfig.mlInfer = ((INPUT_ENABLE_ML && INPUT_ML_INFERENCE_ON) ||
                               (INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_INFERENCE_ON));
}

bool ValidateAndReportEffectiveConfig()
{
   BuildEffectiveConfig();
   Print("EFFECTIVE MATRIX: entry=", (g_effectiveConfig.entry?"ON":"OFF"),
         " close=", (g_effectiveConfig.close?"ON":"OFF"),
         " modSL=", (g_effectiveConfig.modifySL?"ON":"OFF"),
         " modTP=", (g_effectiveConfig.modifyTP?"ON":"OFF"),
         " extreme=", (g_effectiveConfig.extremeRisk?"ON":"OFF"),
         " markov[infer/update]=", (g_effectiveConfig.markovInfer?"ON":"OFF"), "/", (g_effectiveConfig.markovUpdate?"ON":"OFF"),
         " rl[record/learn/infer]=", (g_effectiveConfig.rlRecord?"ON":"OFF"), "/", (g_effectiveConfig.rlLearn?"ON":"OFF"), "/", (g_effectiveConfig.rlInfer?"ON":"OFF"),
         " ml[record/infer]=", (g_effectiveConfig.mlRecord?"ON":"OFF"), "/", (g_effectiveConfig.mlInfer?"ON":"OFF"));

   bool contradiction = false;
   if(!INPUT_TOGGLE_PLACE_ORDERS && (INPUT_TOGGLE_MARKET_ORDERS || INPUT_TOGGLE_PENDING_ORDERS)) contradiction = true;
   if(!INPUT_ENABLE_MARKOV && (INPUT_MARKOV_INFERENCE_ON || INPUT_MARKOV_UPDATE_ON)) contradiction = true;
   if(!INPUT_ENABLE_ML && INPUT_ML_RECORD_ON) contradiction = true;
   if(!INPUT_ENABLE_ML && INPUT_ML_INFERENCE_ON) contradiction = true;
   if(!INPUT_ENABLE_COMBINATION_ADAPTIVE && (INPUT_COMBO_ADAPTIVE_RECORD_ON || INPUT_COMBO_ADAPTIVE_INFERENCE_ON)) contradiction = true;

   if(contradiction)
      Print("WARNING: Effective configuration contradictions detected (master OFF with sub-feature ON).");

   if(INPUT_STRICT_EFFECTIVE_CONFIG_VALIDATION && contradiction)
   {
      Print("STRICT VALIDATION: rejecting init due to contradictory effective config.");
      return false;
   }
   return true;
}

void LogToggleMatrix(const string feature, bool legacyFlag, bool newToggle, bool effective)
{
   Print("TOGGLE MATRIX: ", feature,
         " | legacy=", (legacyFlag ? "ON" : "OFF"),
         " | new=", (newToggle ? "ON" : "OFF"),
         " | effective=", (effective ? "ON" : "OFF"));
}

void BuildDeterministicComboUniverse();
void SaveCombinationStatsSnapshot();
int BuildCanonicalComboSubsets(const string rawCombination, int k, string &subsets[]);
void RebuildDecisionTreeFeatureModule();
int CountSelectedTreeFeatureMatches(const string rawCombination);
double GetTreeConfidenceAdjustment(const string rawCombination);

bool ValidateInputsStrict(string &err)
{
   int totalSignals = MathMin(INPUT_TOTAL_SIGNALS, INPUT_TOTAL_SIGNAL_FACTORS);
   if(INPUT_MAGIC_NUMBER < (int)MAGIC_BASE_MIN || INPUT_MAGIC_NUMBER > (int)MAGIC_BASE_MAX) { err = "INPUT_MAGIC_NUMBER must be within 1..99999999 because EA magic encodes subtype as base + subtype*100000000"; return false; }
   if(INPUT_MAX_CONCURRENT_TRADES < 1) { err = "INPUT_MAX_CONCURRENT_TRADES must be >= 1"; return false; }
   if(INPUT_MAX_SAME_DIRECTION < 1) { err = "INPUT_MAX_SAME_DIRECTION must be >= 1"; return false; }
   if(INPUT_ORDER_COOLDOWN_SECONDS < 0) { err = "INPUT_ORDER_COOLDOWN_SECONDS must be >= 0"; return false; }
   if(INPUT_MAX_DAILY_TRADES < 1) { err = "INPUT_MAX_DAILY_TRADES must be >= 1"; return false; }
   if(INPUT_MAX_CONSECUTIVE_LOSSES < 1) { err = "INPUT_MAX_CONSECUTIVE_LOSSES must be >= 1 when consecutive-loss gate is enabled"; return false; }
   if(!(INPUT_DAILY_LOSS_LIMIT_PERCENT > 0.0 && INPUT_DAILY_LOSS_LIMIT_PERCENT <= 100.0)) { err = "INPUT_DAILY_LOSS_LIMIT_PERCENT must be > 0 and <= 100"; return false; }
   if(!(INPUT_RISK_PERCENT == 0.0 || (INPUT_RISK_PERCENT > 0.0 && INPUT_RISK_PERCENT <= 100.0))) { err = "INPUT_RISK_PERCENT must be 0 or in (0,100]"; return false; }
   if(!(INPUT_RL_WEIGHT >= 0.0 && INPUT_RL_WEIGHT <= 1.0)) { err = "INPUT_RL_WEIGHT must be within [0,1]"; return false; }
   if(!(INPUT_MIN_MTF_SCORE >= 0 && INPUT_MIN_MTF_SCORE <= 10)) { err = "INPUT_MIN_MTF_SCORE must be within 0..10"; return false; }
   if(!(INPUT_MTF_CONSENSUS_VOTE_WEIGHT >= 0.0 && INPUT_MTF_CONSENSUS_VOTE_WEIGHT <= 10.0)) { err = "INPUT_MTF_CONSENSUS_VOTE_WEIGHT must be within [0,10]"; return false; }
   if(!(INPUT_HIGH_SPREAD_CLOSE_PERCENT >= 1.0 && INPUT_HIGH_SPREAD_CLOSE_PERCENT <= 100.0)) { err = "INPUT_HIGH_SPREAD_CLOSE_PERCENT must be within [1,100]"; return false; }
   if(INPUT_SERVER_UTC_OFFSET_HOURS < -14 || INPUT_SERVER_UTC_OFFSET_HOURS > 14) { err = "INPUT_SERVER_UTC_OFFSET_HOURS must be within -14..14"; return false; }
   if(INPUT_MIN_LOT_SIZE <= 0.0) { err = "INPUT_MIN_LOT_SIZE must be > 0"; return false; }
   if(INPUT_MAX_LOT_SIZE < INPUT_MIN_LOT_SIZE) { err = "INPUT_MAX_LOT_SIZE must be >= INPUT_MIN_LOT_SIZE"; return false; }
   if(INPUT_MAX_TOTAL_RISK_PERCENT <= 0.0) { err = "INPUT_MAX_TOTAL_RISK_PERCENT must be > 0"; return false; }
   if(INPUT_PARTIAL_TP_PERCENT < 1.0 || INPUT_PARTIAL_TP_PERCENT > 100.0) { err = "INPUT_PARTIAL_TP_PERCENT must be within 1..100"; return false; }
   if(INPUT_PARTIAL_CLOSE_RATIO <= 0.0 || INPUT_PARTIAL_CLOSE_RATIO > 1.0) { err = "INPUT_PARTIAL_CLOSE_RATIO must be within (0,1]"; return false; }
   if(INPUT_50PCT_TRIGGER_LOW < 0.0 || INPUT_50PCT_TRIGGER_LOW > 100.0) { err = "INPUT_50PCT_TRIGGER_LOW must be within 0..100"; return false; }
   if(INPUT_50PCT_TRIGGER_HIGH < 0.0 || INPUT_50PCT_TRIGGER_HIGH > 100.0) { err = "INPUT_50PCT_TRIGGER_HIGH must be within 0..100"; return false; }
   if(INPUT_50PCT_TRIGGER_LOW > INPUT_50PCT_TRIGGER_HIGH) { err = "INPUT_50PCT_TRIGGER_LOW must be <= INPUT_50PCT_TRIGGER_HIGH"; return false; }
   if(INPUT_TRAIL_ATR_MULTIPLIER <= 0.0) { err = "INPUT_TRAIL_ATR_MULTIPLIER must be > 0"; return false; }
   if(INPUT_TRAIL_STEP_POINTS <= 0.0) { err = "INPUT_TRAIL_STEP_POINTS must be > 0"; return false; }
   if(INPUT_TRAIL_ACTIVATION_POINTS < 0.0) { err = "INPUT_TRAIL_ACTIVATION_POINTS must be >= 0"; return false; }
   if(INPUT_EXECUTION_MODE == PENDING_STOP)
   {
      if(INPUT_PENDING_STOP_OFFSET_POINTS <= 0) { err = "INPUT_PENDING_STOP_OFFSET_POINTS must be > 0 when pending mode is enabled"; return false; }
      if(INPUT_EXEC_PENDING_EXPIRY_ON && INPUT_PENDING_EXPIRY_MINUTES <= 0) { err = "INPUT_PENDING_EXPIRY_MINUTES must be > 0 when pending expiry is enabled"; return false; }
   }
   if(INPUT_POSITION_AGE_HOURS < 0) { err = "INPUT_POSITION_AGE_HOURS must be >= 0"; return false; }
   if(INPUT_TREE_BRANCH_MIN_SUPPORT < 1) { err = "INPUT_TREE_BRANCH_MIN_SUPPORT must be >= 1"; return false; }
   if(INPUT_TREE_MAX_SELECTED_FEATURES < 1) { err = "INPUT_TREE_MAX_SELECTED_FEATURES must be >= 1"; return false; }
   if(INPUT_TREE_MIN_SELECTED_MATCH < 0) { err = "INPUT_TREE_MIN_SELECTED_MATCH must be >= 0"; return false; }
   if(INPUT_MAX_RECOVERY_PER_POS < 0) { err = "INPUT_MAX_RECOVERY_PER_POS must be >= 0"; return false; }
   if(INPUT_RECOVERY_TIMEOUT_MINUTES <= 0) { err = "INPUT_RECOVERY_TIMEOUT_MINUTES must be > 0"; return false; }
   if(INPUT_RECOVERY_TRIGGER_DEPTH < 1.0 || INPUT_RECOVERY_TRIGGER_DEPTH > 95.0) { err = "INPUT_RECOVERY_TRIGGER_DEPTH must be within 1..95"; return false; }
   if(INPUT_RECOVERY_LOT_RATIO_SAFE <= 0 || INPUT_RECOVERY_LOT_RATIO_MOD <= 0 || INPUT_RECOVERY_LOT_RATIO_HIGH <= 0) { err = "All recovery lot ratios must be > 0"; return false; }
   if(INPUT_RECOVERY_LOT_RATIO_SAFE > 10 || INPUT_RECOVERY_LOT_RATIO_MOD > 10 || INPUT_RECOVERY_LOT_RATIO_HIGH > 10) { err = "Recovery lot ratios exceed policy bound (10x)"; return false; }
   if(INPUT_GRID_STEP_POINTS <= 0) { err = "INPUT_GRID_STEP_POINTS must be > 0"; return false; }
   if(INPUT_RECOVERY_MODE == RECOVERY_MARTINGALE && INPUT_MARTINGALE_MULTIPLIER <= 1.0) { err = "INPUT_MARTINGALE_MULTIPLIER must be > 1 in martingale mode"; return false; }
   if(INPUT_RECOVERY_MODE == RECOVERY_HEDGING && INPUT_HEDGE_TRIGGER_OFFSET_POINTS <= 0) { err = "INPUT_HEDGE_TRIGGER_OFFSET_POINTS must be > 0 in hedging mode"; return false; }
   if(INPUT_STRICT_EFFECTIVE_CONFIG_VALIDATION)
   {
      if(!IsValidHourValue(INPUT_ASIAN_START) || !IsValidHourValue(INPUT_ASIAN_END) ||
         !IsValidHourValue(INPUT_LONDON_START) || !IsValidHourValue(INPUT_LONDON_END) ||
         !IsValidHourValue(INPUT_NY_START) || !IsValidHourValue(INPUT_NY_END))
      {
         err = "Session start/end inputs must be valid hours (0..23) when INPUT_STRICT_EFFECTIVE_CONFIG_VALIDATION=true";
         return false;
      }
   }
   totalSignals = MathMin(INPUT_TOTAL_SIGNALS, INPUT_TOTAL_SIGNAL_FACTORS);
   if(totalSignals < 1 || totalSignals > 8) { err = "INPUT_TOTAL_SIGNALS/INPUT_TOTAL_SIGNAL_FACTORS must resolve to 1..8"; return false; }
   if(INPUT_MIN_SIGNALS < 1 || INPUT_MIN_SIGNALS > totalSignals) { err = "INPUT_MIN_SIGNALS must be within 1..total_signals"; return false; }
   return true;
}

//+------------------------------------------------------------------+
//| SECTION 6: INITIALIZATION                                        |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| V7.33: Parse comma-separated string into double array            |
//+------------------------------------------------------------------+
int ParseCSVToArray(string csv, double &outArray[], int maxElements = 10)
{
   string parts[];
   int count = StringSplit(csv, ',', parts);
   
   if(count <= 0 || count > maxElements)
   {
      Print("ERROR: ParseCSVToArray failed. Count=", count, " Max=", maxElements, " CSV=", csv);
      return 0;
   }
   
   for(int i = 0; i < count; i++)
   {
      StringTrimLeft(parts[i]);
      StringTrimRight(parts[i]);
      outArray[i] = StringToDouble(parts[i]);
      
      if(outArray[i] <= 0 || outArray[i] > 100)
      {
         Print("WARNING: Invalid percentage value: ", outArray[i], " at index ", i);
         outArray[i] = MathMax(1.0, MathMin(100.0, outArray[i]));
      }
   }
   
   return count;
}

//+------------------------------------------------------------------+
//| V7.33: Validate and normalize percentage arrays                  |
//+------------------------------------------------------------------+
void ValidatePartialCloseArrays()
{
   // Parse loss percentages and triggers
   g_lossPartsCount = ParseCSVToArray(INPUT_LOSS_PARTS_PERCENTAGES, g_lossPartPercentages, 10);
   int lossTrigCount = ParseCSVToArray(INPUT_LOSS_PARTS_TRIGGERS, g_lossPartTriggers, 10);
   
   if(g_lossPartsCount != lossTrigCount)
   {
      Print("WARNING: Loss percentages count (", g_lossPartsCount, ") != triggers count (", lossTrigCount, ")");
      g_lossPartsCount = MathMin(g_lossPartsCount, lossTrigCount);
   }
   
   if(g_lossPartsCount > INPUT_LOSS_PARTS_COUNT)
   {
      Print("WARNING: Parsed loss parts (", g_lossPartsCount, ") > INPUT_LOSS_PARTS_COUNT (", INPUT_LOSS_PARTS_COUNT, ")");
      g_lossPartsCount = INPUT_LOSS_PARTS_COUNT;
   }
   
   // Parse profit percentages and triggers
   g_profitPartsCount = ParseCSVToArray(INPUT_PROFIT_PARTS_PERCENTAGES, g_profitPartPercentages, 10);
   int profitTrigCount = ParseCSVToArray(INPUT_PROFIT_PARTS_TRIGGERS, g_profitPartTriggers, 10);
   
   if(g_profitPartsCount != profitTrigCount)
   {
      Print("WARNING: Profit percentages count (", g_profitPartsCount, ") != triggers count (", profitTrigCount, ")");
      g_profitPartsCount = MathMin(g_profitPartsCount, profitTrigCount);
   }
   
   if(g_profitPartsCount > INPUT_PROFIT_PARTS_COUNT)
   {
      Print("WARNING: Parsed profit parts (", g_profitPartsCount, ") > INPUT_PROFIT_PARTS_COUNT (", INPUT_PROFIT_PARTS_COUNT, ")");
      g_profitPartsCount = INPUT_PROFIT_PARTS_COUNT;
   }
   
   // Log configuration
   Print("=== V7.33 PARTIAL CLOSE CONFIG ===");
   Print("Loss Parts: ", g_lossPartsCount);
   for(int i = 0; i < g_lossPartsCount; i++)
   {
      Print("  Part ", i+1, ": Close ", g_lossPartPercentages[i], "% at ", g_lossPartTriggers[i], "% loss");
   }
   Print("Profit Parts: ", g_profitPartsCount);
   for(int i = 0; i < g_profitPartsCount; i++)
   {
      Print("  Part ", i+1, ": Close ", g_profitPartPercentages[i], "% at ", g_profitPartTriggers[i], "% profit");
   }
}

int OnInit()
{
   //--- Expiry check
   if(IsEAExpired())
   {
      Print("EA EXPIRED. Please contact support for a new version.");
      Alert("EA V7 HumanBrain has expired!");
      return INIT_FAILED;
   }

   //--- Validate inputs
   string strictErr = "";
   if(!ValidateInputsStrict(strictErr))
   {
      Print("ERROR: ", strictErr);
      return INIT_PARAMETERS_INCORRECT;
   }
   if(INPUT_MIN_SIGNALS < 1 || INPUT_MIN_SIGNALS > 8)
   {
      Print("ERROR: INPUT_MIN_SIGNALS must be 1-8");
      return INIT_PARAMETERS_INCORRECT;
   }
   if(INPUT_MIN_CONFIDENCE < 0 || INPUT_MIN_CONFIDENCE > 100)
   {
      Print("ERROR: INPUT_MIN_CONFIDENCE must be 0-100");
      return INIT_PARAMETERS_INCORRECT;
   }

   if(!g_rngSeeded)
   {
      int seed = (int)(TimeLocal() ^ (datetime)GetTickCount());
      MathSrand(seed);
      g_rngSeeded = true;
      if(INPUT_ENABLE_RL)
         Print("RNG SEEDED: RL random generator initialized once at startup | seed=", seed);
   }

   if(INPUT_GATE_SESSION_WINDOW_ON)
   {
      string sessionErr = "";
      if(!ValidateSessionHourConfig(sessionErr))
      {
         if(INPUT_STRICT_EFFECTIVE_CONFIG_VALIDATION)
         {
            Print("ERROR: ", sessionErr, " | strict session validation enabled.");
            return INIT_PARAMETERS_INCORRECT;
         }

         ValidateSessionHourInputs();
         Print("WARNING: Invalid session window configuration detected: ", sessionErr,
               " | Non-strict fallback enabled; runtime session gate may block all entries.");
      }
   }

   if(INPUT_AI_MODE != AI_OFF && StringLen(INPUT_AI_API_KEY) < 3)
   {
      Print("WARNING: AI mode enabled but API key not configured. AI will be disabled.");
   }
   if(INPUT_AI_MODE != AI_OFF && StringFind(INPUT_AI_API_KEY, "sk-") != 0)
   {
      Print("WARNING: DeepSeek API key should start with 'sk-'. Current key may be invalid.");
   }

   //--- Initialize symbol specs
   g_point        = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
   g_tickSize     = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   g_tickValue    = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   g_lotStep      = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   g_lotDigits    = (int)MathRound(-MathLog10(MathMax(g_lotStep, 1e-8)));
   g_lotDigits    = (int)MathMax(0, MathMin(g_lotDigits, 8));
   g_minLot       = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   g_maxLot       = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   g_digits       = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
    bool invalidVolumeStep = (!MathIsValidNumber(g_lotStep) || g_lotStep <= 0.0);
   bool invalidVolumeMin = (!MathIsValidNumber(g_minLot) || g_minLot <= 0.0);
   bool invalidVolumeMax = (!MathIsValidNumber(g_maxLot) || g_maxLot <= 0.0 || g_maxLot < g_minLot);
   if(invalidVolumeStep || invalidVolumeMin || invalidVolumeMax)
   {
      PrintFormat("=== INVALID SYMBOL VOLUME CONSTRAINTS - INIT FAILED ===\n"
                  "Symbol: %s\n"
                  "SYMBOL_VOLUME_STEP: %.10f\n"
                  "SYMBOL_VOLUME_MIN: %.10f\n"
                  "SYMBOL_VOLUME_MAX: %.10f\n"
                  "Action: Trading disabled. Check broker/tester symbol volume settings.",
                  _Symbol,
                  g_lotStep,
                  g_minLot,
                  g_maxLot);
      return INIT_FAILED;
   }
   g_contractSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
   g_stopLevel    = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
   g_freezeLevel  = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_FREEZE_LEVEL);
    bool invalidTickValue    = (!MathIsValidNumber(g_tickValue)    || g_tickValue <= 0.0);
   bool invalidTickSize     = (!MathIsValidNumber(g_tickSize)     || g_tickSize <= 0.0);
   bool invalidContractSize = (!MathIsValidNumber(g_contractSize) || g_contractSize <= 0.0);
   bool invalidPoint        = (!MathIsValidNumber(g_point)        || g_point <= 0.0);

   //--- Tick value fallback (XAUUSD, indices, etc.) for internal what-if calculations only
   double fallbackTickValue = g_tickValue;
   if(invalidTickValue)
   {
      double testProfit = 0;
      if(OrderCalcProfit(ORDER_TYPE_BUY, _Symbol, 1.0,
         SymbolInfoDouble(_Symbol, SYMBOL_ASK),
         SymbolInfoDouble(_Symbol, SYMBOL_ASK) + g_point, testProfit))
      {
         if(testProfit > 0 && MathIsValidNumber(testProfit))
            fallbackTickValue = testProfit;
         else
            fallbackTickValue = 1.0;
      }
      else
         fallbackTickValue = 1.0;
   }

   if(invalidTickValue || invalidTickSize || invalidContractSize || invalidPoint)
   {
      g_tickValue = fallbackTickValue;
      PrintFormat("=== INVALID SYMBOL ECONOMICS - INIT FAILED ===\n"
                  "Symbol: %s\n"
                  "SYMBOL_TRADE_TICK_VALUE: %.10f (fallback_for_internal_what_if=%.10f)\n"
                  "SYMBOL_TRADE_TICK_SIZE: %.10f\n"
                  "SYMBOL_TRADE_CONTRACT_SIZE: %.10f\n"
                  "SYMBOL_POINT: %.10f\n"
                  "Action: Trading disabled. Check broker/tester symbol settings.",
                  _Symbol,
                  g_tickValue,
                  fallbackTickValue,
                  g_tickSize,
                  g_contractSize,
                  g_point);
      return INIT_FAILED;
   }

   //--- Initialize risk profile
   switch(INPUT_RISK_PROFILE)
   {
      case RISK_LOW:         g_risk.riskPercent = 0.5; break;
      case RISK_MEDIUM:      g_risk.riskPercent = 1.0; break;
      case RISK_MEDIUM_HIGH: g_risk.riskPercent = 1.5; break;
      case RISK_HIGH:        g_risk.riskPercent = 2.0; break;
      case RISK_VERY_HIGH:   g_risk.riskPercent = 3.0; break;
   }
   if(INPUT_RISK_PERCENT > 0)
      g_risk.riskPercent = INPUT_RISK_PERCENT;
   g_risk.maxLot      = INPUT_MAX_LOT_SIZE;
   g_risk.minLot      = INPUT_MIN_LOT_SIZE;
   g_risk.maxTotalRisk = INPUT_MAX_TOTAL_RISK_PERCENT;

   //--- Initialize adaptive parameters
   ResetAdaptiveParamsToDefaults();
   ZeroMemory(g_gateDiagnostics);
   //--- Setup trade object
   g_trade.SetExpertMagicNumber(BuildMagicForSubtype(SUBTYPE_MAIN));
   g_trade.SetDeviationInPoints(30);
  g_selectedFillingMode = GetFillingMode();
   g_trade.SetTypeFilling(g_selectedFillingMode);
   g_trade.SetAsyncMode(false);

   if(INPUT_ENABLE_LOGGING)
      Print("MTF SETTINGS: minScore=", INPUT_MIN_MTF_SCORE,
            " | consensusWeight=", DoubleToString(INPUT_MTF_CONSENSUS_VOTE_WEIGHT, 2));

   if(INPUT_ENABLE_LOGGING)
      Print("FILL MODE SELECTED: ", EnumToString(g_selectedFillingMode));

   //--- Initialize ALL indicator handles
   if(!InitializeIndicators())
   {
      Print("ERROR: Failed to create indicator handles");
      return INIT_FAILED;
   }

   //--- Initialize arrays
   ArrayResize(g_positions, MAX_POSITIONS);
   g_positionCount = 0;
   ArrayResize(g_fingerprints, MAX_FINGERPRINTS);
   g_fingerprintCount = 0;
   ArrayResize(g_trainingData, INPUT_MAX_TRAINING_DATA);
   g_trainingDataCount = 0;
   ArrayResize(g_combinationStats, MAX_COMBINATION_STATS);
   g_combinationStatsCount = 0;
   BuildDeterministicComboUniverse();
   ArrayResize(g_pendingRL, 100);
   g_pendingRLCount = 0;
   ArrayResize(g_recentlyClosedContext, 256);
   g_recentlyClosedContextCount = 0;
   ArrayResize(g_logSuppressionKeys, 0);
   ArrayResize(g_logSuppressionTimes, 0);
   g_logSuppressionCount = 0;
   ArrayResize(g_markovQueue, 0);
   g_markovQueueCount = 0;
   ArrayResize(g_tpModifyFailures, 0);

   //--- Initialize Q-table (all zeros - neutral start)
   ArrayInitialize(g_qTable, 0);
   ArrayInitialize(g_qVisits, 0);

   //--- Initialize Markov transitions (uniform prior)
   for(int i = 0; i < MARKOV_STATES; i++)
   {
      for(int j = 0; j < MARKOV_STATES; j++)
      {
         g_markovTransitions[i][j] = 1.0 / MARKOV_STATES;
         g_markovCounts[i][j] = 1; // Laplace smoothing
      }
   }

   //--- Initialize AI response
   g_aiResponse.marketBias = "neutral";
   g_aiResponse.confidenceScore = 50.0;
   g_aiResponse.riskAlert = false;
   g_aiResponse.lastUpdate = 0;
   g_aiResponse.consecutiveErrors = 0;
   g_lastValidAIResponse = g_aiResponse;
   ArrayResize(g_positionCloseAccumulators, 0);
   g_positionCloseAccumulatorCount = 0;

   if(INPUT_RESET_ALL_PERSISTED_STATE)
   {
      bool resetOk = ResetAllPersistedStateFiles();
      Print("RESET PERSISTENCE ON INIT: ", (resetOk ? "SUCCESS" : "FAILED"));
   }

   //--- Load persisted learning data (only if effective feature paths need it)
   if(INPUT_ENABLE_FINGERPRINT)
      LoadFingerprintData();

   bool needTrainingData = ((INPUT_ENABLE_ML && (INPUT_ML_INFERENCE_ON || INPUT_ML_RECORD_ON)) ||
                            (INPUT_ENABLE_COMBINATION_ADAPTIVE && (INPUT_COMBO_ADAPTIVE_INFERENCE_ON || INPUT_COMBO_ADAPTIVE_RECORD_ON)));
   if(needTrainingData)
   {
      LoadTrainingData();
      // Migration/reset note:
      // Older datasets may have session/day fields attributed from current time instead of deal close time.
      // Set INPUT_RESET_LEGACY_SESSION_DATA=true once to purge stale rows and rebuild clean history.
      if(INPUT_RESET_LEGACY_SESSION_DATA)
      {
         string legacyFile = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_training.csv";
         if(FileIsExist(legacyFile))
            FileDelete(legacyFile);
         g_trainingDataCount = 0;
         Print("MIGRATION: Legacy training dataset reset requested. Historical session/day labels cleared; new records will use close-time attribution.");
      }
      else
      {
         Print("MIGRATION NOTE: If your training set was generated before close-time session/day fix, enable INPUT_RESET_LEGACY_SESSION_DATA once to rebuild statistics.");
      }
      RecalculateCombinationStats();
   }

   if(INPUT_ENABLE_RL)
      LoadQTable();

   bool markovLoadSaveEnabled = (INPUT_ENABLE_MARKOV && (INPUT_MARKOV_INFERENCE_ON || INPUT_MARKOV_UPDATE_ON));
   if(markovLoadSaveEnabled)
      LoadMarkovData();

   if(INPUT_ENABLE_ADAPTIVE)
      LoadAdaptiveParams();

   if(INPUT_ENABLE_COMBINATION_ADAPTIVE && !INPUT_ENABLE_ML)
      Print("WARNING: Combination-adaptive enabled while ML disabled. Using training-data only mode.");

   LoadRuntimeState();

   g_effExtremeByThreat = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_BY_THREAT);
   g_effExtremeByDrawdown = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_BY_DRAWDOWN);
   g_effExtremeHysteresisExit = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_HYSTERESIS_EXIT);
   g_effDrawdownProtectState = ResolveRuntimeToggle(false, INPUT_ENABLE_DRAWDOWN_PROTECT_STATE);
   g_effExtremeOnTickHandler = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_ON_TICK_HANDLER);
   g_effExtremeOnTickEarlyReturn = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_ON_TICK_EARLY_RETURN);
   g_effExtremeCloseOldest = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_CLOSE_OLDEST);
   g_effExtremeFilterSymbol = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_FILTER_SYMBOL);
   g_effExtremeFilterMagic = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_FILTER_MAGIC);
   g_effExtremeThrottle = ResolveRuntimeToggle(false, INPUT_ENABLE_EXTREME_THROTTLE);
   g_effEquityFloorTrigger = ResolveRuntimeToggle(false, INPUT_ENABLE_EQUITY_FLOOR_TRIGGER);
   g_effEquityFloorForceState = ResolveRuntimeToggle(false, INPUT_ENABLE_EQUITY_FLOOR_FORCE_EXTREME_STATE);
   g_effEquityFloorCloseAll = ResolveRuntimeToggle(false, INPUT_ENABLE_EQUITY_FLOOR_CLOSE_ALL);
   g_effEquityFloorReturn = ResolveRuntimeToggle(false, INPUT_ENABLE_EQUITY_FLOOR_RETURN_AFTER_ACTION);
   g_effCloseAllApi = ResolveRuntimeToggle(false, INPUT_ENABLE_CLOSE_ALL_POSITIONS_API);
   g_effCloseAllOnlyOur = ResolveRuntimeToggle(false, INPUT_ENABLE_CLOSE_ALL_ONLY_OUR_POSITIONS);
   g_effCloseAllSymbolFilter = ResolveRuntimeToggle(false, INPUT_ENABLE_CLOSE_ALL_SYMBOL_FILTER);
   g_effGateProtectionBlock = ResolveRuntimeToggle(false, INPUT_ENABLE_GATE_BLOCK_ON_PROTECTION_STATE);
   g_effThreatHardBlock = ResolveRuntimeToggle(false, INPUT_ENABLE_THREAT_HARD_BLOCK);
   g_effThreatExtremeZoneBlock = ResolveRuntimeToggle(false, INPUT_ENABLE_THREAT_EXTREME_ZONE_BLOCK);
   g_effThreatSoftLotShrink = ResolveRuntimeToggle(false, INPUT_ENABLE_THREAT_SOFT_LOT_SHRINK);
   g_effCloseRecoveryTimeout = ResolveRuntimeToggle(false, INPUT_ENABLE_CLOSE_RECOVERY_TIMEOUT);
   g_effClosePositionAgeTimeout = ResolveRuntimeToggle(INPUT_POSITION_AGE_HOURS > 0, INPUT_ENABLE_CLOSE_POSITION_AGE_TIMEOUT);
   g_effCloseHighSpreadProfit = ResolveRuntimeToggle(INPUT_CLOSE_PROFIT_ON_HIGH_SPREAD, INPUT_ENABLE_CLOSE_HIGH_SPREAD_PROFIT);
   g_effClose50PctDefensive = ResolveRuntimeToggle(INPUT_ENABLE_50PCT_CLOSE, INPUT_ENABLE_CLOSE_50PCT_DEFENSIVE);
   g_effClosePartialTP = ResolveRuntimeToggle(INPUT_ENABLE_PARTIAL_CLOSE, INPUT_ENABLE_CLOSE_PARTIAL_TP);
   g_effCloseMultiLevelPartial = ResolveRuntimeToggle(false, INPUT_ENABLE_CLOSE_MULTI_LEVEL_PARTIAL);
   g_effModifyMoveToBE = ResolveRuntimeToggle(INPUT_MOVE_BE_AFTER_PARTIAL, INPUT_ENABLE_MODIFY_MOVE_TO_BREAKEVEN);
   g_effModifyTrailingSL = ResolveRuntimeToggle(INPUT_ENABLE_TRAILING, INPUT_ENABLE_MODIFY_TRAILING_SL);
   g_effModifyTrailingTP = ResolveRuntimeToggle(INPUT_ENABLE_TRAILING_TP, INPUT_ENABLE_MODIFY_TRAILING_TP);
   g_effModifySkipLossOnHighSpread = ResolveRuntimeToggle(INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD, INPUT_ENABLE_MODIFY_SKIP_LOSS_ON_HIGH_SPREAD);

   LogToggleMatrix("UpdateEAState.ExtremeByThreat", false, INPUT_ENABLE_EXTREME_BY_THREAT, g_effExtremeByThreat);
   LogToggleMatrix("UpdateEAState.ExtremeByDrawdown", false, INPUT_ENABLE_EXTREME_BY_DRAWDOWN, g_effExtremeByDrawdown);
   LogToggleMatrix("UpdateEAState.HysteresisExit", false, INPUT_ENABLE_EXTREME_HYSTERESIS_EXIT, g_effExtremeHysteresisExit);
   LogToggleMatrix("UpdateEAState.DrawdownProtectState", false, INPUT_ENABLE_DRAWDOWN_PROTECT_STATE, g_effDrawdownProtectState);
   LogToggleMatrix("OnTick.ExtremeHandler", false, INPUT_ENABLE_EXTREME_ON_TICK_HANDLER, g_effExtremeOnTickHandler);
   LogToggleMatrix("OnTick.ExtremeEarlyReturn", false, INPUT_ENABLE_EXTREME_ON_TICK_EARLY_RETURN, g_effExtremeOnTickEarlyReturn);
   LogToggleMatrix("HandleExtremeRisk.CloseOldest", false, INPUT_ENABLE_EXTREME_CLOSE_OLDEST, g_effExtremeCloseOldest);
   LogToggleMatrix("EquityFloor.Trigger", false, INPUT_ENABLE_EQUITY_FLOOR_TRIGGER, g_effEquityFloorTrigger);
   LogToggleMatrix("CloseAllPositions.API", false, INPUT_ENABLE_CLOSE_ALL_POSITIONS_API, g_effCloseAllApi);
   LogToggleMatrix("CheckAllGates.ProtectionState", false, INPUT_ENABLE_GATE_BLOCK_ON_PROTECTION_STATE, g_effGateProtectionBlock);
   LogToggleMatrix("RunDecisionPipeline.ThreatHardBlock", false, INPUT_ENABLE_THREAT_HARD_BLOCK, g_effThreatHardBlock);
   LogToggleMatrix("RunDecisionPipeline.ThreatExtremeZone", false, INPUT_ENABLE_THREAT_EXTREME_ZONE_BLOCK, g_effThreatExtremeZoneBlock);
   LogToggleMatrix("CheckRecoveryTimeouts", false, INPUT_ENABLE_CLOSE_RECOVERY_TIMEOUT, g_effCloseRecoveryTimeout);
   LogToggleMatrix("CheckPositionAgeTimeout", INPUT_POSITION_AGE_HOURS > 0, INPUT_ENABLE_CLOSE_POSITION_AGE_TIMEOUT, g_effClosePositionAgeTimeout);
   LogToggleMatrix("HandleHighSpreadOpenPositions", INPUT_CLOSE_PROFIT_ON_HIGH_SPREAD, INPUT_ENABLE_CLOSE_HIGH_SPREAD_PROFIT, g_effCloseHighSpreadProfit);
   LogToggleMatrix("Handle50PercentLotClose", INPUT_ENABLE_50PCT_CLOSE, INPUT_ENABLE_CLOSE_50PCT_DEFENSIVE, g_effClose50PctDefensive);
   LogToggleMatrix("ManagePartialClose", INPUT_ENABLE_PARTIAL_CLOSE, INPUT_ENABLE_CLOSE_PARTIAL_TP, g_effClosePartialTP);
   LogToggleMatrix("HandleMultiLevelPartial", false, INPUT_ENABLE_CLOSE_MULTI_LEVEL_PARTIAL, g_effCloseMultiLevelPartial);
   LogToggleMatrix("ManageTrailingStops", INPUT_ENABLE_TRAILING, INPUT_ENABLE_MODIFY_TRAILING_SL, g_effModifyTrailingSL);
   LogToggleMatrix("ManageTrailingTP", INPUT_ENABLE_TRAILING_TP, INPUT_ENABLE_MODIFY_TRAILING_TP, g_effModifyTrailingTP);
   LogToggleMatrix("MoveToBreakeven", INPUT_MOVE_BE_AFTER_PARTIAL, INPUT_ENABLE_MODIFY_MOVE_TO_BREAKEVEN, g_effModifyMoveToBE);
   LogToggleMatrix("ShouldSkipStopAdjustmentsForTicket", INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD, INPUT_ENABLE_MODIFY_SKIP_LOSS_ON_HIGH_SPREAD, g_effModifySkipLossOnHighSpread);

   if(INPUT_USE_LEGACY_BEHAVIOR_MAPPING && INPUT_TOGGLE_RESOLUTION_MODE == TOGGLE_RESOLUTION_MIGRATION)
   {
      bool legacyOverrideFound = false;
      if((INPUT_POSITION_AGE_HOURS > 0) && !INPUT_ENABLE_CLOSE_POSITION_AGE_TIMEOUT) legacyOverrideFound = true;
      if(INPUT_CLOSE_PROFIT_ON_HIGH_SPREAD && !INPUT_ENABLE_CLOSE_HIGH_SPREAD_PROFIT) legacyOverrideFound = true;
      if(INPUT_ENABLE_50PCT_CLOSE && !INPUT_ENABLE_CLOSE_50PCT_DEFENSIVE) legacyOverrideFound = true;
      if(INPUT_ENABLE_PARTIAL_CLOSE && !INPUT_ENABLE_CLOSE_PARTIAL_TP) legacyOverrideFound = true;
      if(INPUT_MOVE_BE_AFTER_PARTIAL && !INPUT_ENABLE_MODIFY_MOVE_TO_BREAKEVEN) legacyOverrideFound = true;
      if(INPUT_ENABLE_TRAILING && !INPUT_ENABLE_MODIFY_TRAILING_SL) legacyOverrideFound = true;
      if(INPUT_ENABLE_TRAILING_TP && !INPUT_ENABLE_MODIFY_TRAILING_TP) legacyOverrideFound = true;
      if(INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD && !INPUT_ENABLE_MODIFY_SKIP_LOSS_ON_HIGH_SPREAD) legacyOverrideFound = true;
      if(legacyOverrideFound)
         Print("WARNING: Legacy mapping override active (legacy=true + new=false found). Consider TOGGLE_RESOLUTION_NEW_AUTH.");
   }

   if(!ValidateAndReportEffectiveConfig())
      return INIT_FAILED;

   Print("CHECKLIST GUARD UpdateEAState=", (g_effExtremeByThreat || g_effExtremeByDrawdown || g_effDrawdownProtectState ? "ON" : "OFF"));
   Print("CHECKLIST GUARD OnTick extreme branch=", ((g_effExtremeOnTickHandler || g_effExtremeOnTickEarlyReturn) ? "ON" : "OFF"));
   Print("CHECKLIST GUARD HandleExtremeRisk=", (g_effExtremeCloseOldest ? "ON" : "OFF"));
   Print("CHECKLIST GUARD Equity floor block=", (g_effEquityFloorTrigger ? "ON" : "OFF"));
   Print("CHECKLIST GUARD CloseAllPositions=", (g_effCloseAllApi ? "ON" : "OFF"));
   Print("CHECKLIST GUARD CheckAllGates protection-state gate=", (g_effGateProtectionBlock ? "ON" : "OFF"));
   Print("CHECKLIST GUARD RunDecisionPipeline threat blocks=", ((g_effThreatHardBlock || g_effThreatExtremeZoneBlock || g_effThreatSoftLotShrink) ? "ON" : "OFF"));
   Print("CHECKLIST GUARD CheckRecoveryTimeouts=", (g_effCloseRecoveryTimeout ? "ON" : "OFF"));
   Print("CHECKLIST GUARD CheckPositionAgeTimeout=", (g_effClosePositionAgeTimeout ? "ON" : "OFF"));
   Print("CHECKLIST GUARD HandleHighSpreadOpenPositions=", (g_effCloseHighSpreadProfit ? "ON" : "OFF"));
   Print("CHECKLIST GUARD Handle50PercentLotClose=", (g_effClose50PctDefensive ? "ON" : "OFF"));
   Print("CHECKLIST GUARD ManagePartialClose=", (g_effClosePartialTP ? "ON" : "OFF"));
   Print("CHECKLIST GUARD HandleMultiLevelPartial=", (g_effCloseMultiLevelPartial ? "ON" : "OFF"));
   Print("CHECKLIST GUARD ManageTrailingStops=", (g_effModifyTrailingSL ? "ON" : "OFF"));
   Print("CHECKLIST GUARD ManageTrailingTP=", (g_effModifyTrailingTP ? "ON" : "OFF"));
   Print("CHECKLIST GUARD MoveToBreakeven=", (g_effModifyMoveToBE ? "ON" : "OFF"));
   Print("CHECKLIST GUARD ShouldSkipStopAdjustmentsForTicket=", (g_effModifySkipLossOnHighSpread ? "ON" : "OFF"));

   //--- Master toggle compatibility matrix
   Print("FEATURE MATRIX [Placement]: place=", (INPUT_TOGGLE_PLACE_ORDERS?"ON":"OFF"),
         " market=", (INPUT_TOGGLE_MARKET_ORDERS?"ON":"OFF"),
         " pending=", (INPUT_TOGGLE_PENDING_ORDERS?"ON":"OFF"));
   Print("FEATURE MATRIX [Close]: close=", (INPUT_TOGGLE_CLOSE_ORDERS?"ON":"OFF"),
         " equityFloor=", (INPUT_CLOSE_EQUITY_FLOOR_ON?"ON":"OFF"),
         " partialTP=", (INPUT_CLOSE_PARTIAL_TP_ON?"ON":"OFF"));
   Print("FEATURE MATRIX [Modify]: SL=", (INPUT_TOGGLE_MODIFY_STOPS?"ON":"OFF"),
         " TP=", (INPUT_TOGGLE_MODIFY_TPS?"ON":"OFF"),
         " brokerGuard=", (INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON?"ON":"OFF"));
   Print("FEATURE MATRIX [Learning]: RL[infer/learn]=", (INPUT_ENABLE_RL && INPUT_RL_INFERENCE_ON?"ON":"OFF"), "/", (INPUT_ENABLE_RL && INPUT_RL_LEARNING_ON?"ON":"OFF"),
         " Markov[infer/update]=", (INPUT_ENABLE_MARKOV && INPUT_MARKOV_INFERENCE_ON?"ON":"OFF"), "/", (INPUT_ENABLE_MARKOV && INPUT_MARKOV_UPDATE_ON?"ON":"OFF"),
         " ML[infer/record]=", (INPUT_ENABLE_ML && INPUT_ML_INFERENCE_ON?"ON":"OFF"), "/", (INPUT_ENABLE_ML && INPUT_ML_RECORD_ON?"ON":"OFF"),
         " Combo[infer/record]=", (INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_INFERENCE_ON?"ON":"OFF"), "/", (INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_RECORD_ON?"ON":"OFF"),
         " AI=", (INPUT_AI_MODE != AI_OFF && INPUT_AI_QUERY_ON?"ON":"OFF"));

   if(!INPUT_TOGGLE_PLACE_ORDERS && (INPUT_TOGGLE_MARKET_ORDERS || INPUT_TOGGLE_PENDING_ORDERS))
      Print("WARNING: Placement master OFF overrides market/pending sub-toggles.");
   if(!INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON)
      Print("WARNING: Broker-distance modify guard is OFF (diagnostics only; unsafe live).");

   //--- Initialize daily stats
   g_startingBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   g_peakEquity      = AccountInfoDouble(ACCOUNT_EQUITY);
   ResetDailyCounters();

   //--- Sync existing positions - FIXED: Reset count first
   g_positionCount = 0;
   SyncExistingPositions();

   //--- Calculate initial average ATR
   CalculateAverageATR();

   Print("=== EA "+EA_VERSION_LABEL+" HumanBrain Complete - INITIALIZED ===");
   Print("Symbol: ", _Symbol, " | Magic: ", INPUT_MAGIC_NUMBER);
   Print("Risk: ", g_risk.riskPercent, "% | MaxTrades: ", g_adaptive.maxPositions);
      Print("ENTRY POLICY: closeOnOpposite=", (INPUT_CLOSE_ON_OPPOSITE_SIGNAL ? "ON" : "OFF"),
         " | strictFlip=", (INPUT_STRICT_OPPOSITE_FLIP_MODE ? "ON" : "OFF"),
         " | hardCap=", (INPUT_MAX_MAIN_HARD_CAP_ON ? "ON" : "OFF"),
         " | cancelOppPending=", (INPUT_FLIP_CANCEL_OPPOSITE_PENDING_ON ? "ON" : "OFF"),
         " | flipBypassCooldown=", (INPUT_FLIP_BYPASS_COOLDOWN_ON ? "ON" : "OFF"),
         " | adaptiveMaxExpansion=", (INPUT_ALLOW_ADAPTIVE_MAX_POSITION_EXPANSION ? "ON" : "OFF"),
         " | inputMaxMain=", INPUT_MAX_CONCURRENT_TRADES);
   Print("Q-Learning: ", INPUT_ENABLE_RL ? "ON" : "OFF", " | Markov: ", INPUT_ENABLE_MARKOV ? "ON" : "OFF");
   Print("ML: ", INPUT_ENABLE_ML ? "ON" : "OFF", " | AI: ", EnumToString(INPUT_AI_MODE));
   Print("Adaptive: ", INPUT_ENABLE_ADAPTIVE ? "ON" : "OFF");
   Print("Stop Level: ", g_stopLevel, " | Freeze Level: ", g_freezeLevel);
   Print("Min Signals: ", INPUT_MIN_SIGNALS, " | Min Confidence: ", INPUT_MIN_CONFIDENCE, "%");
   Print("FEATURES: RL=", (INPUT_ENABLE_RL?"ON":"OFF"),
         " Markov=", (INPUT_ENABLE_MARKOV?"ON":"OFF"),
         " ML=", (INPUT_ENABLE_ML?"ON":"OFF"),
         " ComboAdaptive=", (INPUT_ENABLE_COMBINATION_ADAPTIVE?"ON":"OFF"),
         " Adaptive=", (INPUT_ENABLE_ADAPTIVE?"ON":"OFF"),
         " Recovery=", (INPUT_ENABLE_RECOVERY?"ON":"OFF"),
         " Partial=", (INPUT_ENABLE_PARTIAL_CLOSE?"ON":"OFF"),
         " 50pct=", (INPUT_ENABLE_50PCT_CLOSE?"ON":"OFF"),
         " TrailSL=", (INPUT_ENABLE_TRAILING?"ON":"OFF"),
         " TrailTP=", (INPUT_ENABLE_TRAILING_TP?"ON":"OFF"));

   EventSetTimer(AI_TIMER_SECONDS);
   return INIT_SUCCEEDED;
}
//+------------------------------------------------------------------+
bool ValidateIndicatorHandle(int handle, const string name, bool required)
{
   if(!required)
      return true;

   if(handle != INVALID_HANDLE)
      return true;

   Print("ERROR: Required indicator handle invalid: ", name);
   return false;
}
//+------------------------------------------------------------------+
bool InitializeIndicators()
{
   // M1 indicators
   g_hEmaFast_M1  = iMA(_Symbol, PERIOD_M1, INPUT_EMA_FAST, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaSlow_M1  = iMA(_Symbol, PERIOD_M1, INPUT_EMA_SLOW, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaTrend_M1 = iMA(_Symbol, PERIOD_M1, INPUT_EMA_TREND, 0, MODE_EMA, PRICE_CLOSE);
   g_hRSI_M1      = iRSI(_Symbol, PERIOD_M1, INPUT_RSI_PERIOD, PRICE_CLOSE);
   g_hStoch_M1    = iStochastic(_Symbol, PERIOD_M1, INPUT_STOCH_K, INPUT_STOCH_D, INPUT_STOCH_SLOW, MODE_SMA, STO_LOWHIGH);
   g_hMACD_M1     = iMACD(_Symbol, PERIOD_M1, INPUT_MACD_FAST, INPUT_MACD_SLOW, INPUT_MACD_SIGNAL, PRICE_CLOSE);
   g_hWPR_M1      = iWPR(_Symbol, PERIOD_M1, INPUT_WPR_PERIOD);
   g_hATR_M1      = iATR(_Symbol, PERIOD_M1, INPUT_ATR_PERIOD);
   g_hADX_M1      = iADX(_Symbol, PERIOD_M1, INPUT_ADX_PERIOD);
   g_hBB_M1       = iBands(_Symbol, PERIOD_M1, INPUT_BB_PERIOD, 0, INPUT_BB_DEVIATION, PRICE_CLOSE);
   g_hVolume_M1   = iVolumes(_Symbol, PERIOD_M1, VOLUME_TICK);

   // M5 indicators
   g_hEmaFast_M5  = iMA(_Symbol, PERIOD_M5, INPUT_EMA_FAST, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaSlow_M5  = iMA(_Symbol, PERIOD_M5, INPUT_EMA_SLOW, 0, MODE_EMA, PRICE_CLOSE);
   g_hATR_M5      = iATR(_Symbol, PERIOD_M5, INPUT_ATR_PERIOD);

   // H1 indicators
   g_hEmaFast_H1  = iMA(_Symbol, PERIOD_H1, INPUT_EMA_FAST, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaSlow_H1  = iMA(_Symbol, PERIOD_H1, INPUT_EMA_SLOW, 0, MODE_EMA, PRICE_CLOSE);
   g_hATR_H1      = iATR(_Symbol, PERIOD_H1, INPUT_ATR_PERIOD);
   g_hADX_H1      = iADX(_Symbol, PERIOD_H1, INPUT_ADX_PERIOD);

   // H4 indicators
   g_hEmaFast_H4  = iMA(_Symbol, PERIOD_H4, INPUT_EMA_FAST, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaSlow_H4  = iMA(_Symbol, PERIOD_H4, INPUT_EMA_SLOW, 0, MODE_EMA, PRICE_CLOSE);

   // D1 indicators
   g_hEmaFast_D1  = iMA(_Symbol, PERIOD_D1, INPUT_EMA_FAST, 0, MODE_EMA, PRICE_CLOSE);
   g_hEmaSlow_D1  = iMA(_Symbol, PERIOD_D1, INPUT_EMA_SLOW, 0, MODE_EMA, PRICE_CLOSE);

   bool allValid = true;

   // Core M1 indicators (always required)
   allValid &= ValidateIndicatorHandle(g_hEmaFast_M1,  "M1 EMA Fast", true);
   allValid &= ValidateIndicatorHandle(g_hEmaSlow_M1,  "M1 EMA Slow", true);
   allValid &= ValidateIndicatorHandle(g_hEmaTrend_M1, "M1 EMA Trend", true);
   allValid &= ValidateIndicatorHandle(g_hRSI_M1,      "M1 RSI", true);
   allValid &= ValidateIndicatorHandle(g_hStoch_M1,    "M1 Stochastic", true);
   allValid &= ValidateIndicatorHandle(g_hMACD_M1,     "M1 MACD", true);
   allValid &= ValidateIndicatorHandle(g_hWPR_M1,      "M1 Williams %R", true);
   allValid &= ValidateIndicatorHandle(g_hATR_M1,      "M1 ATR", true);
   allValid &= ValidateIndicatorHandle(g_hADX_M1,      "M1 ADX", true);
   allValid &= ValidateIndicatorHandle(g_hBB_M1,       "M1 Bollinger Bands", true);
   allValid &= ValidateIndicatorHandle(g_hVolume_M1,   "M1 Volume", true);

   // M5/H1/H4/D1 handles used by MTF, regime, and SL/TP logic
   allValid &= ValidateIndicatorHandle(g_hEmaFast_M5,  "M5 EMA Fast", true);
   allValid &= ValidateIndicatorHandle(g_hEmaSlow_M5,  "M5 EMA Slow", true);
   allValid &= ValidateIndicatorHandle(g_hATR_M5,      "M5 ATR", true);
   allValid &= ValidateIndicatorHandle(g_hEmaFast_H1,  "H1 EMA Fast", true);
   allValid &= ValidateIndicatorHandle(g_hEmaSlow_H1,  "H1 EMA Slow", true);
   allValid &= ValidateIndicatorHandle(g_hATR_H1,      "H1 ATR", true);
   allValid &= ValidateIndicatorHandle(g_hADX_H1,      "H1 ADX", true);
   allValid &= ValidateIndicatorHandle(g_hEmaFast_H4,  "H4 EMA Fast", true);
   allValid &= ValidateIndicatorHandle(g_hEmaSlow_H4,  "H4 EMA Slow", true);
   allValid &= ValidateIndicatorHandle(g_hEmaFast_D1,  "D1 EMA Fast", true);
   allValid &= ValidateIndicatorHandle(g_hEmaSlow_D1,  "D1 EMA Slow", true);

   // Optional feature-gated validations can be added here for ML/RL extras.

   return allValid;
}
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   EventKillTimer();

   //--- Save all learning data
   if(INPUT_ENABLE_FINGERPRINT)
      SaveFingerprintData();
   bool needTrainingDataPersist = ((INPUT_ENABLE_ML && (INPUT_ML_INFERENCE_ON || INPUT_ML_RECORD_ON)) ||
                                   (INPUT_ENABLE_COMBINATION_ADAPTIVE && (INPUT_COMBO_ADAPTIVE_INFERENCE_ON || INPUT_COMBO_ADAPTIVE_RECORD_ON)));
   if(needTrainingDataPersist)
      SaveTrainingData();
   SaveCombinationStatsSnapshot();
   if(INPUT_ENABLE_RL)
      SaveQTable();
   if(INPUT_ENABLE_MARKOV && (INPUT_MARKOV_INFERENCE_ON || INPUT_MARKOV_UPDATE_ON))
      SaveMarkovData();
   if(INPUT_ENABLE_ADAPTIVE)
      SaveAdaptiveParams();

   SaveRuntimeState();

   //--- Release indicator handles
   ReleaseIndicators();

   //--- Remove chart objects
   ObjectsDeleteAll(0, "V7_");

   if((bool)MQLInfoInteger(MQL_TESTER))
   {
      int openPositions = CountAllOurPositions();
      if(openPositions > 0)
      {
         double balance = AccountInfoDouble(ACCOUNT_BALANCE);
         double equity = AccountInfoDouble(ACCOUNT_EQUITY);
         double openPL = GetOpenProfitLoss();
         Print("TEST END NOTICE: ", openPositions,
               " position(s) still open. Balance may appear unchanged until forced close. ",
               "Balance=", DoubleToString(balance, 2),
               " Equity=", DoubleToString(equity, 2),
               " OpenP/L=", DoubleToString(openPL, 2));
      }
   }

      Print("Gate diagnostics summary | session=", g_gateDiagnostics.sessionRejects,
         " cooldown=", g_gateDiagnostics.cooldownRejects,
         " signals=", g_gateDiagnostics.signalsRejects,
         " mtf=", g_gateDiagnostics.mtfRejects,
          " mtf_data=", g_gateDiagnostics.mtfDataReadRejects,
         " adx_data=", g_gateDiagnostics.adxDataReadRejects,
         " threat=", g_gateDiagnostics.threatRejects,
         " confidence=", g_gateDiagnostics.confidenceRejects,
         " max_positions=", g_gateDiagnostics.maxPositionsRejects);
   Print("=== EA "+EA_VERSION_LABEL+" HumanBrain DEINITIALIZED (reason=", reason, ") ===");
}
//+------------------------------------------------------------------+
void ReleaseIndicators()
{
   if(g_hEmaFast_M1 != INVALID_HANDLE) IndicatorRelease(g_hEmaFast_M1);
   if(g_hEmaSlow_M1 != INVALID_HANDLE) IndicatorRelease(g_hEmaSlow_M1);
   if(g_hEmaTrend_M1 != INVALID_HANDLE) IndicatorRelease(g_hEmaTrend_M1);
   if(g_hRSI_M1 != INVALID_HANDLE) IndicatorRelease(g_hRSI_M1);
   if(g_hStoch_M1 != INVALID_HANDLE) IndicatorRelease(g_hStoch_M1);
   if(g_hMACD_M1 != INVALID_HANDLE) IndicatorRelease(g_hMACD_M1);
   if(g_hWPR_M1 != INVALID_HANDLE) IndicatorRelease(g_hWPR_M1);
   if(g_hATR_M1 != INVALID_HANDLE) IndicatorRelease(g_hATR_M1);
   if(g_hADX_M1 != INVALID_HANDLE) IndicatorRelease(g_hADX_M1);
   if(g_hBB_M1 != INVALID_HANDLE) IndicatorRelease(g_hBB_M1);
   if(g_hVolume_M1 != INVALID_HANDLE) IndicatorRelease(g_hVolume_M1);
   if(g_hEmaFast_M5 != INVALID_HANDLE) IndicatorRelease(g_hEmaFast_M5);
   if(g_hEmaSlow_M5 != INVALID_HANDLE) IndicatorRelease(g_hEmaSlow_M5);
   if(g_hATR_M5 != INVALID_HANDLE) IndicatorRelease(g_hATR_M5);
   if(g_hEmaFast_H1 != INVALID_HANDLE) IndicatorRelease(g_hEmaFast_H1);
   if(g_hEmaSlow_H1 != INVALID_HANDLE) IndicatorRelease(g_hEmaSlow_H1);
   if(g_hATR_H1 != INVALID_HANDLE) IndicatorRelease(g_hATR_H1);
   if(g_hADX_H1 != INVALID_HANDLE) IndicatorRelease(g_hADX_H1);
   if(g_hEmaFast_H4 != INVALID_HANDLE) IndicatorRelease(g_hEmaFast_H4);
   if(g_hEmaSlow_H4 != INVALID_HANDLE) IndicatorRelease(g_hEmaSlow_H4);
   if(g_hEmaFast_D1 != INVALID_HANDLE) IndicatorRelease(g_hEmaFast_D1);
   if(g_hEmaSlow_D1 != INVALID_HANDLE) IndicatorRelease(g_hEmaSlow_D1);
}
//+------------------------------------------------------------------+
//| SECTION 7: Expiry check                                          |
//+------------------------------------------------------------------+
bool IsEAExpired()
{
   datetime expiryDate = StringToTime(IntegerToString(INPUT_EXPIRY_YEAR) + "." +
                         IntegerToString(INPUT_EXPIRY_MONTH) + "." +
                         IntegerToString(INPUT_EXPIRY_DAY));
   return (TimeCurrent() >= expiryDate);
}
//+------------------------------------------------------------------+
//| SECTION 8: OnTick - MAIN HANDLER                                 |
//+------------------------------------------------------------------+
void OnTick()
{
   ulong tickStartMs = GetTickCount();
   datetime now = TimeCurrent();
   RefreshSymbolTradeConstraints();

   static bool expiryWarned = false;
   bool expired = IsEAExpired();
   if(expired && !expiryWarned)
   {
      Print("EA EXPIRED. No new trades will be placed.");
      expiryWarned = true;
   }

   SyncPositionStates();

   ulong t0 = GetTickCount();
   int adaptiveIntervalSec = MathMax(1, INPUT_HEAVY_BASE_INTERVAL_SECONDS);
   if(g_lastHeavyHistoryRun == 0 || (now - g_lastHeavyHistoryRun) >= adaptiveIntervalSec)
   {
      ProcessClosedPositions();
      ProcessEntryDeals();
      CleanupInactivePositions();
      CleanupRecentClosedContext();
      CleanupStalePendingRL();
      g_lastHeavyHistoryRun = now;
   }
   g_tickMsHistory = GetTickCount() - t0;

   HandleHighSpreadOpenPositions();

   if(INPUT_ENABLE_ADAPTIVE)
      CheckAdaptiveOptimization();

   t0 = GetTickCount();
   ManageTrailingTP();
   CheckPositionAgeTimeout();
   CleanupExpiredPendingStopOrders();
   ManageTrailingStops();
   ManagePartialClose();
   for(int i = 0; i < PositionsTotal(); i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(g_effCloseMultiLevelPartial)
         HandleMultiLevelPartial(ticket);
   }
   if(g_effClose50PctDefensive)
      Handle50PercentLotClose();
   if(INPUT_ENABLE_RECOVERY && !expired && INPUT_RECOVERY_MODE != RECOVERY_OFF)
   {
      if(INPUT_RECOVERY_MODE == RECOVERY_AVERAGING) MonitorRecoveryAveragingMode();
      else if(INPUT_RECOVERY_MODE == RECOVERY_HEDGING) MonitorRecoveryHedgingMode();
      else if(INPUT_RECOVERY_MODE == RECOVERY_GRID) MonitorRecoveryGridMode();
      else if(INPUT_RECOVERY_MODE == RECOVERY_MARTINGALE) MonitorRecoveryMartingaleMode();
   }
   CheckRecoveryTimeouts();
   g_tickMsManagePositions = GetTickCount() - t0;

   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   if(equity > g_peakEquity) g_peakEquity = equity;

   if(INPUT_CLOSE_EQUITY_FLOOR_ON && g_effEquityFloorTrigger)
   {
      double equityFloor = g_startingBalance * (INPUT_EQUITY_FLOOR_PERCENT / 100.0);
      if(equity < equityFloor)
      {
         bool acted = false;
         Print("EQUITY FLOOR BREACH DETECTED: ", equity, " < ", equityFloor);
         if(g_effEquityFloorForceState)
         {
            g_eaState = STATE_EXTREME_RISK;
            acted = true;
         }
         if(g_effEquityFloorCloseAll)
         {
            CloseAllPositions("EQUITY_FLOOR");
            acted = true;
         }
         if(!acted && ShouldPrintOncePerWindow("equity_floor_actions_disabled", 60))
            Print("EQUITY FLOOR WARNING: breach detected but all configured actions are disabled.");
         if(g_effEquityFloorReturn)
            return;
      }
   }

   CheckDailyReset();
   UpdateEAState();

   if(g_prevEaState == STATE_EXTREME_RISK && g_eaState != STATE_EXTREME_RISK)
   {
      if(ShouldPrintOncePerWindow("extreme_risk_resolved", 60))
      {
         g_lastExtremeResolvedLogTime = TimeCurrent();
         Print("Extreme risk resolved. Resuming normal operation.");
      }
   }

   if(g_eaState == STATE_EXTREME_RISK)
   {
      if(g_effExtremeOnTickHandler)
         HandleExtremeRisk();
      else if(ShouldPrintOncePerWindow("extreme_handler_disabled", 60))
         Print("EXTREME_ON_TICK: HandleExtremeRisk disabled (EXTREME_HANDLER_DISABLED)");

      if(g_effExtremeOnTickEarlyReturn)
         return;
      if(ShouldPrintOncePerWindow("extreme_early_return_disabled", 60))
         Print("EXTREME_ON_TICK: early return disabled (EXTREME_EARLY_RETURN_DISABLED)");
   }

   t0 = GetTickCount();
   datetime currentBarTime = iTime(_Symbol, PERIOD_M1, 0);
   if(currentBarTime != g_lastBarTime)
   {
      g_lastBarTime = currentBarTime;
      UpdateAverageSpread();
      CalculateAverageATR();
      DetectMarketRegime();
      if(!expired)
      {
          DecisionResult decision;
         ZeroMemory(decision);
         if(RunDecisionPipeline(decision) && decision.shouldTrade)
         {
            bool canExecuteOrder = true;
            bool hasOppositeOpen = false;
            bool hasOppositePending = false;

            for(int i = PositionsTotal() - 1; i >= 0; i--)
            {
               ulong posTicket = PositionGetTicket(i);
               if(posTicket == 0 || !PositionSelectByTicket(posTicket) || !IsOurPosition(posTicket))
                  continue;
               string posComment = PositionGetString(POSITION_COMMENT);
               if(StringFind(posComment, COMMENT_MAIN_PREFIX) < 0 || !IsMainEntryComment(posComment))
                  continue;
               int posDir = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
               if(posDir != decision.direction)
               {
                  hasOppositeOpen = true;
                  break;
               }
            }

            if(!hasOppositeOpen)
            
            {
               int oppositePending = CountMainPendingStopsByDirection(decision.direction * -1);
               hasOppositePending = (oppositePending > 0);
            }
            bool isFlipEvent = (hasOppositeOpen || hasOppositePending);
            if(INPUT_CLOSE_ON_OPPOSITE_SIGNAL)
            {
             if(!CleanupOppositeExposureForFlip(decision.direction))
               {
                  canExecuteOrder = false;
                  if(INPUT_ENABLE_LOGGING)
                     LogWithRestartGuard("ENTRY BLOCKED: opposite cleanup failed");
         
            }
         }
          g_currentEntryIsFlip = (INPUT_STRICT_OPPOSITE_FLIP_MODE && isFlipEvent);
            g_flipCooldownBypassLogged = false;
            if(INPUT_ENABLE_LOGGING)
               Print("ENTRY CONTEXT: ", (g_currentEntryIsFlip ? "flip" : "normal"),
                     " | direction=", (decision.direction == 1 ? "BUY" : "SELL"));

            if(canExecuteOrder)
               ExecuteOrder(decision);

            g_currentEntryIsFlip = false;
            }
      }
   }
   g_tickMsDecision = GetTickCount() - t0;

   t0 = GetTickCount();
   if(INPUT_SHOW_PANEL && (g_lastPanelRun == 0 || (now - g_lastPanelRun) >= 2))
   {
      DrawStatsPanel();
      g_lastPanelRun = now;
   }
   g_tickMsPanel = GetTickCount() - t0;

   t0 = GetTickCount();
   if(INPUT_STATE_CHECKPOINT_MINUTES > 0 &&
      (g_lastCheckpointTime == 0 || now - g_lastCheckpointTime >= INPUT_STATE_CHECKPOINT_MINUTES * 60) &&
      (int)(GetTickCount() - tickStartMs) <= INPUT_ON_TICK_BUDGET_MS)
   {
      SaveRuntimeState();
      if(INPUT_ENABLE_FINGERPRINT)
         SaveFingerprintData();
      g_lastCheckpointTime = now;
   }
   g_tickMsPersistence = GetTickCount() - t0;
}

bool CloseMainPositionsOppositeToSignal(int direction)
{
   bool allClosed = true;
   int profitableClosedCount = 0;
   int skippedLosingCount = 0;
   
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0 || !PositionSelectByTicket(ticket) || !IsOurPosition(ticket))
         continue;

      string comment = PositionGetString(POSITION_COMMENT);
      if(StringFind(comment, COMMENT_MAIN_PREFIX) < 0)
         continue;
      if(!IsMainEntryComment(comment))
         continue;

      int posType = (int)PositionGetInteger(POSITION_TYPE);
      int posDir = (posType == POSITION_TYPE_BUY) ? 1 : -1;
      
      // Skip positions in the same direction as new signal
      if(posDir == direction)
         continue;

      // ===== V7.32 FIX: CHECK PROFIT BEFORE CLOSING =====
      // Get current profit/loss of the position
      double positionProfit = PositionGetDouble(POSITION_PROFIT);
      
      // Only close if position is in PROFIT (above 0)
      if(positionProfit > 0)
      {
         if(!g_trade.PositionClose(ticket))
         {
            allClosed = false;
            Print("FLIP_CLEANUP CLOSE FAILED: ticket=", ticket,
                  " | profit=", DoubleToString(positionProfit, 2),
                  " | retcode=", g_trade.ResultRetcode(),
                  " | comment=", g_trade.ResultComment());
         }
         else
         {
            profitableClosedCount++;
            Print("FLIP_CLEANUP CLOSE (PROFIT): closed ticket=", ticket,
                  " | profit=", DoubleToString(positionProfit, 2),
                  " | oldDir=", (posDir == 1 ? "BUY" : "SELL"),
                  " | newDir=", (direction == 1 ? "BUY" : "SELL"));
         }
      }
      else
      {
         // Position is in loss or breakeven - keep it running
         skippedLosingCount++;
         Print("FLIP_CLEANUP SKIP (LOSS/BE): kept ticket=", ticket,
               " | profit=", DoubleToString(positionProfit, 2),
               " | direction=", (posDir == 1 ? "BUY" : "SELL"),
               " | reason=NOT_PROFITABLE");
      }
      // ===== END OF V7.32 FIX =====
   }
   
   Print("FLIP_CLEANUP SUMMARY: profitableClosed=", profitableClosedCount,
         " | skippedLosing=", skippedLosingCount);
   
   return allClosed;
}


bool CancelMainPendingStopsOppositeToDirection(int direction)
{
   bool allCanceled = true;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      ulong ticket = OrderGetTicket(i);
      if(ticket == 0 || !OrderSelect(ticket))
         continue;
      if(!IsOurMainPendingStopOrder(ticket))
         continue;

      ENUM_ORDER_TYPE type = (ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
      int orderDir = (type == ORDER_TYPE_BUY_STOP) ? 1 : -1;
      if(orderDir == direction)
         continue;

      MqlTradeRequest request;
      MqlTradeResult result;
      ZeroMemory(request);
      ZeroMemory(result);

      request.action = TRADE_ACTION_REMOVE;
      request.order = ticket;
      request.symbol = _Symbol;
      request.magic = BuildMagicForSubtype(SUBTYPE_MAIN);
      request.comment = "FLIP_CANCEL_OPPOSITE_PENDING";

      bool sent = OrderSend(request, result);
      if(sent && result.retcode == TRADE_RETCODE_DONE)
      {
         RemovePendingRLByOrderTicket(ticket);
         Print("FLIP_CLEANUP CANCEL_PENDING: ticket=", ticket,
               " | type=", EnumToString(type),
               " | retcode=", result.retcode);
      }
      else
      {
         allCanceled = false;
         Print("FLIP_CLEANUP CANCEL_PENDING FAILED: ticket=", ticket,
               " | type=", EnumToString(type),
               " | retcode=", result.retcode,
               " | comment=", result.comment);
      }
   }
   return allCanceled;
}

bool CleanupOppositeExposureForFlip(int direction)
{
   if(g_flipCleanupInProgress)
      return true;

   g_flipCleanupInProgress = true;
   bool allOk = true;

   int openMainBefore = CountMainPositionsFromBroker();
   int pendingBuyBefore = CountMainPendingStopsByDirection(1);
   int pendingSellBefore = CountMainPendingStopsByDirection(-1);
   Print("FLIP_CLEANUP PRE: targetDir=", (direction == 1 ? "BUY" : "SELL"),
         " | openMain=", openMainBefore,
         " | pendingBuy=", pendingBuyBefore,
         " | pendingSell=", pendingSellBefore);

   if(!CloseMainPositionsOppositeToSignal(direction))
      allOk = false;

   if(INPUT_FLIP_CANCEL_OPPOSITE_PENDING_ON)
   {
      if(!CancelMainPendingStopsOppositeToDirection(direction))
         allOk = false;
   }

   int openMainAfter = CountMainPositionsFromBroker();
   int pendingBuyAfter = CountMainPendingStopsByDirection(1);
   int pendingSellAfter = CountMainPendingStopsByDirection(-1);
   Print("FLIP_CLEANUP POST: targetDir=", (direction == 1 ? "BUY" : "SELL"),
         " | openMain=", openMainAfter,
         " | pendingBuy=", pendingBuyAfter,
         " | pendingSell=", pendingSellAfter,
         " | status=", (allOk ? "OK" : "FAILED"));

   g_flipCleanupInProgress = false;
   return allOk;
}

void OnTimer()
{
   if(INPUT_AI_MODE == AI_OFF || !INPUT_AI_QUERY_ON)
      return;

   ulong timerStart = GetTickCount();
   QueryDeepSeekAI();
   g_tickMsAIRequest = GetTickCount() - timerStart;
}

void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
   if(trans.type != TRADE_TRANSACTION_DEAL_ADD)
      return;
   if(trans.deal == 0)
      return;
   if(!HistoryDealSelect(trans.deal))
      return;

   if((string)HistoryDealGetString(trans.deal, DEAL_SYMBOL) != _Symbol)
      return;

   long magic = HistoryDealGetInteger(trans.deal, DEAL_MAGIC);
   if(!IsOurMagic(magic))
      return;

   long entryType = HistoryDealGetInteger(trans.deal, DEAL_ENTRY);
   if(entryType != DEAL_ENTRY_IN)
      return;

   if(trans.order == 0 || !HistoryOrderSelect(trans.order))
      return;
   ENUM_ORDER_TYPE activatedOrderType = (ENUM_ORDER_TYPE)HistoryOrderGetInteger(trans.order, ORDER_TYPE);
   if(activatedOrderType != ORDER_TYPE_BUY_STOP && activatedOrderType != ORDER_TYPE_SELL_STOP)
      return;

   long dealType = HistoryDealGetInteger(trans.deal, DEAL_TYPE);
   int activatedDirection = 0;
   if(dealType == DEAL_TYPE_BUY)
      activatedDirection = 1;
   else if(dealType == DEAL_TYPE_SELL)
      activatedDirection = -1;
   else
      return;

   if(INPUT_CLOSE_ON_OPPOSITE_SIGNAL)
   {
      bool ok = CleanupOppositeExposureForFlip(activatedDirection);
      Print("FLIP_ON_ACTIVATION: direction=", (activatedDirection == 1 ? "BUY" : "SELL"),
            " | cleanup=", (ok ? "OK" : "FAILED"),
            " | deal=", trans.deal,
            " | order=", trans.order,
            " | reqAction=", request.action,
            " | resultRetcode=", result.retcode);
   }
}

//+------------------------------------------------------------------+
//| SECTION 9: EA STATE MANAGEMENT                                   |
//+------------------------------------------------------------------+
void UpdateEAState()
{
   ENUM_EA_STATE previousState = g_eaState;

   int mainCount = CountMainPositionsFromBroker();
   int totalCount = CountAllOurPositions();
   double threat = CalculateMarketThreat();
   double drawdownPct = CalculateDrawdownPercent();

   bool causeThreat = (g_effExtremeByThreat && threat > INPUT_EXTREME_ENTER_THREAT);
   bool causeDrawdown = (g_effExtremeByDrawdown && drawdownPct > INPUT_EXTREME_ENTER_DRAWDOWN);
   bool enterExtreme = (causeThreat || causeDrawdown);

   bool canExitExtreme = (threat < INPUT_EXTREME_EXIT_THREAT &&
                          drawdownPct < INPUT_EXTREME_EXIT_DRAWDOWN &&
                          totalCount <= INPUT_EXTREME_EXIT_MAX_TOTAL_POSITIONS);

   if(g_effExtremeHysteresisExit)
   {
      if(previousState == STATE_EXTREME_RISK)
      {
         if(!canExitExtreme)
         {
            g_eaState = STATE_EXTREME_RISK;
            g_prevEaState = previousState;
            return;
         }
      }
      else if(enterExtreme)
      {
         g_eaState = STATE_EXTREME_RISK;
         g_prevEaState = previousState;
         return;
      }
   }
   else
   {
      if(enterExtreme)
      {
         g_eaState = STATE_EXTREME_RISK;
         g_prevEaState = previousState;
         return;
      }
   }

   if(g_effDrawdownProtectState && drawdownPct > INPUT_EXTREME_EXIT_DRAWDOWN)
      g_eaState = STATE_DRAWDOWN_PROTECT;
   else if(CountRecoveryPositions() > 0)
      g_eaState = STATE_RECOVERY_ACTIVE;
   else if(Count50PctReducedPositions() > 0)
      g_eaState = STATE_POSITION_REDUCED;
   else if(mainCount > 0)
      g_eaState = STATE_POSITION_ACTIVE;
   else
      g_eaState = STATE_IDLE;

   g_prevEaState = previousState;
}
//+------------------------------------------------------------------+
void HandleExtremeRisk()
{
   static datetime lastCloseAttempt = 0;
   if(!g_effExtremeCloseOldest)
   {
      if(ShouldPrintOncePerWindow("extreme_close_oldest_disabled", 60))
         Print("EXTREME_RISK: no-op (EXTREME_CLOSE_OLDEST_DISABLED)");
      return;
   }

   if(g_effExtremeThrottle)
   {
      int intervalSec = MathMax(1, INPUT_EXTREME_CLOSE_INTERVAL_SECONDS);
      if(TimeCurrent() - lastCloseAttempt < intervalSec)
      {
         if(ShouldPrintOncePerWindow("extreme_throttle_active", 60))
            Print("EXTREME_RISK: throttle active (EXTREME_THROTTLE_ACTIVE)");
         return;
      }
   }
   else if(ShouldPrintOncePerWindow("extreme_throttle_disabled", 60))
      Print("EXTREME_RISK: throttle disabled (EXTREME_THROTTLE_DISABLED)");

   lastCloseAttempt = TimeCurrent();

   int maxCloses = MathMax(1, INPUT_EXTREME_MAX_CLOSES_PER_CALL);
   int closedCount = 0;
   for(int k = 0; k < maxCloses; k++)
   {
      ulong oldestTicket = 0;
      datetime oldestTime = TimeCurrent();

      int total = PositionsTotal();
      for(int i = 0; i < total; i++)
      {
         ulong ticket = PositionGetTicket(i);
         if(ticket == 0) continue;
         if(!PositionSelectByTicket(ticket)) continue;

         if(g_effExtremeFilterSymbol)
         {
            if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
         }
         else if(ShouldPrintOncePerWindow("extreme_filter_symbol_disabled", 60))
            Print("EXTREME_RISK: symbol filter disabled (EXTREME_FILTER_SYMBOL_DISABLED)");

         if(g_effExtremeFilterMagic)
         {
            if(!IsOurMagic(PositionGetInteger(POSITION_MAGIC))) continue;
         }
         else if(ShouldPrintOncePerWindow("extreme_filter_magic_disabled", 60))
            Print("EXTREME_RISK: magic filter disabled (EXTREME_FILTER_MAGIC_DISABLED)");

         datetime posTime = (datetime)PositionGetInteger(POSITION_TIME);
         if(posTime < oldestTime)
         {
            oldestTime = posTime;
            oldestTicket = ticket;
         }
      }

      if(oldestTicket == 0)
         break;

      if(g_trade.PositionClose(oldestTicket))
      {
         closedCount++;
         Print("EXTREME RISK: Closed oldest position ", oldestTicket, " (EXTREME_CLOSE_OLDEST)");
      }
      else
      {
         Print("EXTREME RISK: Close failed for ", oldestTicket, " (EXTREME_CLOSE_OLDEST_FAILED)");
         break;
      }
   }

   if(closedCount == 0 && ShouldPrintOncePerWindow("extreme_no_candidates", 60))
      Print("EXTREME_RISK: no matching positions to close (EXTREME_NO_CANDIDATES)");
}
//+------------------------------------------------------------------+
//| SECTION 10: POSITION COUNTING - FIXED!                           |
//+------------------------------------------------------------------+
uint FNV1aStart() { return 2166136261; }
uint FNV1aUpdateByte(uint hash, uint b) { return (hash ^ (b & 0xFF)) * 16777619; }
uint FNV1aUpdateInt(uint hash, int value)
{
   uint v=(uint)value;
   for(int i=0;i<4;i++) hash=FNV1aUpdateByte(hash,(v>>(i*8))&0xFF);
   return hash;
}
uint FNV1aUpdateLong(uint hash, long value)
{
   ulong v=(ulong)value;
   for(int i=0;i<8;i++) hash=FNV1aUpdateByte(hash,(uint)((v>>(i*8))&0xFF));
   return hash;
}
uint FNV1aUpdateDouble(uint hash, double value)
{
   long scaled=(long)MathRound(value*1000000.0);
   return FNV1aUpdateLong(hash, scaled);
}

int BuildMagicForSubtype(ENUM_POSITION_SUBTYPE subtype)
{
   long subtypeValue = (long)((int)subtype);
   if(subtypeValue < 0 || subtypeValue > 9)
   {
      Print("MAGIC ERROR: invalid subtype value=", subtypeValue, " | fallback to base magic");
      return (int)INPUT_MAGIC_NUMBER;
   }

   long composed = (long)INPUT_MAGIC_NUMBER + (subtypeValue * MAGIC_SUBTYPE_MULTIPLIER);
   if(composed <= 0 || composed > 2147483647)
   {
      Print("MAGIC ERROR: overflow composing magic from base=", INPUT_MAGIC_NUMBER,
            " subtype=", subtypeValue, " | fallback to base magic");
      return (int)INPUT_MAGIC_NUMBER;
   }
   return (int)composed;
}
bool IsOurMagic(const long magic)
{
   if(magic <= 0)
      return false;

   long subtype = magic / MAGIC_SUBTYPE_MULTIPLIER;
   if(subtype < 0 || subtype > 9)
      return false;

   long base = magic - (subtype * MAGIC_SUBTYPE_MULTIPLIER);
   if(base < MAGIC_BASE_MIN || base > MAGIC_BASE_MAX)
      return false;

   return (base == (long)INPUT_MAGIC_NUMBER);
}
ENUM_POSITION_SUBTYPE InferSubtypeFromComment(const string &comment)
{
   if(StringFind(comment, COMMENT_RECOVERY_PREFIX) >= 0) return SUBTYPE_RECOVERY;
   if(StringFind(comment, COMMENT_AVG_PREFIX) >= 0) return SUBTYPE_AVERAGING;
   return SUBTYPE_MAIN;
}
bool IsAuxSubtypeByMagic(long magic)
{
   if(!IsOurMagic(magic))
      return false;

   long subtype=(magic/MAGIC_SUBTYPE_MULTIPLIER);
   return (subtype==SUBTYPE_RECOVERY || subtype==SUBTYPE_AVERAGING || subtype==SUBTYPE_AUX);
}

//--- Helper: returns true only if the position belongs to this EA
bool IsOurPosition(ulong ticket)
{
   if(!PositionSelectByTicket(ticket))
      return false;
   if(PositionGetString(POSITION_SYMBOL) != _Symbol)
      return false;
   if(!IsOurMagic(PositionGetInteger(POSITION_MAGIC)))
      return false;
   return true;
}
//--- Helper: returns true only if the pending order belongs to this EA
bool IsOurPendingOrder(ulong ticket)
{
   if(!OrderSelect(ticket))
      return false;
   if(OrderGetString(ORDER_SYMBOL) != _Symbol)
      return false;
   if(!IsOurMagic(OrderGetInteger(ORDER_MAGIC)))
      return false;

   ENUM_ORDER_TYPE type = (ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
   if(type != ORDER_TYPE_BUY_STOP && type != ORDER_TYPE_SELL_STOP)
      return false;

   return true;
}
bool IsMainEntryComment(const string &comment)
{
   if(StringFind(comment, COMMENT_RECOVERY_PREFIX) >= 0) return false;
   if(StringFind(comment, COMMENT_AVG_PREFIX) >= 0) return false;
   if(StringFind(comment, COMMENT_HEDGE_PREFIX) >= 0) return false;
   if(StringFind(comment, COMMENT_GRID_PREFIX) >= 0) return false;
   if(StringFind(comment, COMMENT_50PCT_PREFIX) >= 0) return false;
   return true;
}

bool IsOurMainPendingStopOrder(ulong ticket)
{
   if(!IsOurPendingOrder(ticket))
      return false;

   long magic = OrderGetInteger(ORDER_MAGIC);
   if(IsAuxSubtypeByMagic(magic))
      return false;

   string comment = OrderGetString(ORDER_COMMENT);
   return IsMainEntryComment(comment);
}

int CountMainPendingStopsByDirection(int direction)
{
   int count = 0;
   int total = OrdersTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = OrderGetTicket(i);
      if(ticket == 0) continue;
       if(!IsOurMainPendingStopOrder(ticket)) continue;

      ENUM_ORDER_TYPE type = (ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
      int orderDir = (type == ORDER_TYPE_BUY_STOP) ? 1 : -1;
      if(orderDir == direction)
         count++;
   }
   return count;
}

int CountMainPendingStopsAllDirections()
{
   return CountMainPendingStopsByDirection(1) + CountMainPendingStopsByDirection(-1);
}

//--- Count existing pending stop orders by direction (1=BUY, -1=SELL)
int CountPendingStopsByDirection(int direction)
{
   return CountMainPendingStopsByDirection(direction);
}

int GetEffectiveMainExposureCount()
{
   int openMain = CountMainPositionsFromBroker();
   if(INPUT_EXECUTION_MODE == PENDING_STOP && IsFeatureEnabled("pending_orders"))
      return (openMain + CountMainPendingStopsAllDirections());
   return openMain;
}
//--- Remove expired pending stop orders and log event
void CleanupExpiredPendingStopOrders()
{
   if(!INPUT_PENDING_EXPIRY_CLEANUP_ON)
      return;
   int total = OrdersTotal();
   datetime now = TimeCurrent();
   for(int i = total - 1; i >= 0; i--)
   {
      ulong ticket = OrderGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPendingOrder(ticket)) continue;

      datetime expiry = (datetime)OrderGetInteger(ORDER_TIME_EXPIRATION);
      if(expiry <= 0 || expiry > now)
         continue;

      ENUM_ORDER_TYPE type = (ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
      Print("PENDING STOP EXPIRED: Ticket=", ticket,
            " | Type=", EnumToString(type),
            " | Expiry=", TimeToString(expiry, TIME_DATE|TIME_SECONDS),
            " | Now=", TimeToString(now, TIME_DATE|TIME_SECONDS));

      MqlTradeRequest request;
      MqlTradeResult result;
      ZeroMemory(request);
      ZeroMemory(result);

      request.action = TRADE_ACTION_REMOVE;
      request.order = ticket;
      request.symbol = _Symbol;
      request.magic = BuildMagicForSubtype(SUBTYPE_MAIN);
      request.comment = "PENDING_EXPIRED_CLEANUP";

      bool sent = OrderSend(request, result);
      if(sent && result.retcode == TRADE_RETCODE_DONE)
      {
         RemovePendingRLByOrderTicket(ticket);
         Print("PENDING STOP CANCELED: Ticket=", ticket,
               " | Type=", EnumToString(type),
               " | Reason=Expired");
      }
      else
      {
         Print("PENDING STOP CANCEL FAILED: Ticket=", ticket,
               " | Retcode=", result.retcode,
               " | Comment=", result.comment);
      }
   }
}
//--- Count only MAIN positions (exclude recovery/avg/hedge/etc.)
int CountMainPositionsFromBroker()
{
   int mainCount = 0;
   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      long magic = PositionGetInteger(POSITION_MAGIC);
      string comment = PositionGetString(POSITION_COMMENT);
      bool auxByMagic = IsAuxSubtypeByMagic(magic);
      bool auxByComment = (StringFind(comment, COMMENT_RECOVERY_PREFIX) >= 0 ||
                           StringFind(comment, COMMENT_AVG_PREFIX)      >= 0 ||
                           StringFind(comment, COMMENT_HEDGE_PREFIX)    >= 0 ||
                           StringFind(comment, COMMENT_GRID_PREFIX)     >= 0 ||
                           StringFind(comment, COMMENT_50PCT_PREFIX)    >= 0);
      if(auxByMagic || auxByComment) continue;

      mainCount++;
   }
   return mainCount;
}
//--- Count **all** positions belonging to this EA (main + any aux)
int CountAllOurPositions()
{
   int count = 0;
   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      count++;
   }
   return count;
}
//--- Sum floating P/L for all positions belonging to this EA
double GetOpenProfitLoss()
{
   double openProfitLoss = 0.0;
   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      openProfitLoss += PositionGetDouble(POSITION_PROFIT);
   }
   return openProfitLoss;
}
//--- Count recovery/averaging positions (used for state machine)
int CountRecoveryPositions()
{
   int count = 0;
   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      long magic = PositionGetInteger(POSITION_MAGIC);
      string comment = PositionGetString(POSITION_COMMENT);
      if(IsAuxSubtypeByMagic(magic) ||
         StringFind(comment, COMMENT_RECOVERY_PREFIX) >= 0 ||
         StringFind(comment, COMMENT_AVG_PREFIX)      >= 0)
         count++;
   }
   return count;
}
//--- Count positions that already performed the 50%?lot close
int Count50PctReducedPositions()
{
   int count = 0;
   for(int i = 0; i < g_positionCount; i++)
   {
      if(g_positions[i].isActive && g_positions[i].lotReduced)
         count++;
   }
   return count;
}
//+------------------------------------------------------------------+
//| SECTION 11: 9-FACTOR THREAT ASSESSMENT (Part 5 of Strategy)      |
//+------------------------------------------------------------------+
double CalculateMarketThreat()
{
   double threat = 0;

   //--- FACTOR 1: Position Loss Count (0-75 points, 15 per losing position)
   int losingCount = 0;
   int totalPositions = 0;
   double totalUnrealizedLoss = 0;

   int total = PositionsTotal();
   for(int p = 0; p < total; p++)
   {
      ulong ticket = PositionGetTicket(p);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      totalPositions++;
      double profit = PositionGetDouble(POSITION_PROFIT);
      if(profit < 0)
      {
         losingCount++;
         totalUnrealizedLoss += MathAbs(profit);
      }
   }
   // V7.2 FIX (BUG 7): Reduced from 15 pts/position (max 75) to 8 pts (max 40)
   if(INPUT_THREAT_FACTOR_LOSING_COUNT_ON)
      threat += MathMin(losingCount * 8.0, 40.0);

   // V7.2 FIX (BUG 8): Only apply majority check with 3+ positions (1 losing out of 1 = 100% but meaningless)
   //--- FACTOR 2: Majority Losing (+25 if >50% positions losing, only with 3+ positions)
   if(INPUT_THREAT_FACTOR_MAJORITY_LOSING_ON && totalPositions >= 3 && (double)losingCount / totalPositions > 0.5)
      threat += 25.0;

   //--- FACTOR 3: Account Drawdown (graduated: 0-45 points)
   double drawdownPct = CalculateDrawdownPercent();
   if(INPUT_THREAT_FACTOR_DRAWDOWN_ON && drawdownPct >= 10.0)     threat += 45.0;
   else if(INPUT_THREAT_FACTOR_DRAWDOWN_ON && drawdownPct >= 7.0) threat += 30.0;
   else if(INPUT_THREAT_FACTOR_DRAWDOWN_ON && drawdownPct >= 4.0) threat += 15.0;
   else if(INPUT_THREAT_FACTOR_DRAWDOWN_ON && drawdownPct >= 2.0) threat += 5.0;
   else if(INPUT_THREAT_FACTOR_DRAWDOWN_ON && drawdownPct >= 1.0) threat += 3.0;

   //--- FACTOR 4: Consecutive Loss Streak (non-linear: 0-25 points)
   if(INPUT_THREAT_FACTOR_CONSECUTIVE_LOSS_ON && g_consecutiveLosses >= 5)      threat += 20.0 + (g_consecutiveLosses - 5) * 2;
   else if(INPUT_THREAT_FACTOR_CONSECUTIVE_LOSS_ON && g_consecutiveLosses >= 4) threat += 15.0;
   else if(INPUT_THREAT_FACTOR_CONSECUTIVE_LOSS_ON && g_consecutiveLosses >= 3) threat += 8.0;
   else if(INPUT_THREAT_FACTOR_CONSECUTIVE_LOSS_ON && g_consecutiveLosses >= 2) threat += 3.0;

   //--- FACTOR 5: Volatility Spike (ATR ratio: 0-30 points)
   double volRatio = CalculateVolatilityRatio();
   if(INPUT_THREAT_FACTOR_VOLATILITY_RATIO_ON && volRatio >= 2.0)      threat += 30.0;
   else if(INPUT_THREAT_FACTOR_VOLATILITY_RATIO_ON && volRatio >= 1.6) threat += 20.0;
   else if(INPUT_THREAT_FACTOR_VOLATILITY_RATIO_ON && volRatio >= 1.4) threat += 12.0;
   else if(INPUT_THREAT_FACTOR_VOLATILITY_RATIO_ON && volRatio >= 1.2) threat += 5.0;

   //--- FACTOR 6: News Event Proximity (0-25 points)
   // Simplified: Use time?based heuristic for major news windows
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   // NFP first Friday of month around 13:30 UTC
   if(INPUT_THREAT_FACTOR_NEWS_WINDOW_ON && dt.day_of_week == 5 && dt.day <= 7 && dt.hour >= 13 && dt.hour <= 15)
      threat += 20.0;
   // ECB/Fed typical announcement times
   if(INPUT_THREAT_FACTOR_NEWS_WINDOW_ON && dt.hour >= 12 && dt.hour <= 14 && (dt.day_of_week == 3 || dt.day_of_week == 4))
      threat += 8.0;

   //--- FACTOR 7: Calendar Liquidity Effects (0-15 points)
   // Rationale:
   // - Removed Sunday penalty because many brokers have shortened/reopened sessions with unstable timestamps,
   //   which can over-penalize valid setups.
   // - Friday penalty is now limited to late UTC hours when liquidity often deteriorates and spreads widen.
   // - End-of-month penalty remains configurable with a lower default to reduce over-filtering.
   if(INPUT_THREAT_FRIDAY_LATE_PENALTY_ON && dt.day_of_week == 5 && dt.hour >= INPUT_FRIDAY_LATE_HOUR_UTC)
      threat += INPUT_FRIDAY_LATE_PENALTY;

   if(INPUT_ENABLE_END_OF_MONTH_PENALTY && INPUT_THREAT_END_OF_MONTH_PENALTY_ON && dt.day >= INPUT_END_OF_MONTH_START_DAY)
      threat += INPUT_END_OF_MONTH_PENALTY;

   //--- FACTOR 8: Recovery Order Presence (capped to avoid outsized impact)
   int recoveryCount = CountRecoveryPositions();
   if(INPUT_THREAT_FACTOR_RECOVERY_POSITION_ON && recoveryCount > 0)
      threat += MathMin(recoveryCount * 4.0 + 2.0, 15.0);

   //--- FACTOR 9: Win Streak Complacency
   if(INPUT_THREAT_FACTOR_WIN_STREAK_ON && g_consecutiveWins >= 5)
      threat += 5.0;  // Complacency warning
   else if(INPUT_THREAT_FACTOR_WIN_STREAK_ON && g_consecutiveWins >= 2)
      threat -= 2.0;  // Slight confidence boost

   //--- Apply adaptive multiplier
   threat *= g_adaptive.threatMultiplier;

   //--- Clamp to 0-100
   if(threat < 0) threat = 0;
   if(threat > 100) threat = 100;

   return threat;
}
//+------------------------------------------------------------------+
ENUM_THREAT_ZONE GetThreatZone(double threat)
{
   if(threat >= 81) return THREAT_EXTREME;
   if(threat >= 61) return THREAT_RED;
   if(threat >= 41) return THREAT_ORANGE;
   if(threat >= 21) return THREAT_YELLOW;
   return THREAT_GREEN;
}
//+------------------------------------------------------------------+
double CalculateDrawdownPercent()
{
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   if(g_peakEquity <= 0) return 0;
   return ((g_peakEquity - equity) / g_peakEquity) * 100.0;
}
//+------------------------------------------------------------------+
double CalculateVolatilityRatio()
{
   double atr[];
   if(CopyBuffer(g_hATR_M1, 0, 0, 1, atr) < 1 || atr[0] <= 0)
      return 1.0;

   if(g_averageATR <= 0)
      return 1.0;

   return atr[0] / g_averageATR;
}
//+------------------------------------------------------------------+
void CalculateAverageATR()
{
   double atr[];
   ArraySetAsSeries(atr, true);
   if(CopyBuffer(g_hATR_M5, 0, 0, 100, atr) < 50)
      return;

   double sum = 0;
   for(int i = 0; i < 50; i++)
      sum += atr[i];

   g_averageATR = sum / 50.0;
}
//+------------------------------------------------------------------+
//| SECTION 12: 6-COMPONENT CONFIDENCE CALCULATION (Part 6)          |
//+------------------------------------------------------------------+
double CalculateConfidence(const SignalResult &signals, int direction, int mtfScore,
                            const string &fpId, const string &combination, double threat)
{
   //--- BASE CONFIDENCE: 50% (neutral starting point)
   double conf = 50.0;

   //--- COMPONENT 1: TREND STRENGTH (0 to +35)
   double trendComponent = CalculateTrendStrengthComponent(direction);
   conf += trendComponent;

   //--- COMPONENT 2: MOMENTUM CONFIRMATION (0 to +21)
   double momentumComponent = CalculateMomentumComponent(direction);
   conf += momentumComponent;

   //--- COMPONENT 3: SUPPORT/RESISTANCE LEVELS (-8 to +8)
   double srComponent = CalculateSRComponent(direction);
   conf += srComponent;

   //--- COMPONENT 4: VOLATILITY ADJUSTMENT (-10 to 0) - FIXED: Reduced penalty
   double volComponent = CalculateVolatilityComponent();
   conf += volComponent * 0.5; // FIXED: Halved impact

   //--- COMPONENT 5: TIME/SESSION ANALYSIS (0 to +13)
   double timeComponent = CalculateTimeComponent();
   conf += timeComponent;

   //--- COMPONENT 6: DIVERGENCE/PATTERN DETECTION (0 to +14)
   double divergenceComponent = CalculateDivergenceComponent(direction);
   conf += divergenceComponent;

   //--- FIXED: Add signal count bonus (+5 per signal above minimum)
   int extraSignals = signals.totalSignals - INPUT_MIN_SIGNALS;
   if(extraSignals > 0)
      conf += extraSignals * 5.0;

   //--- Clamp raw confidence
   if(conf < 0) conf = 0;
   if(conf > 100) conf = 100;

   bool mlApplied = false;
   bool mlNonNeutral = false;
   double appliedMLMultiplier = 1.0;
   bool comboApplied = false;
   bool comboNonNeutral = false;
   double appliedComboMultiplier = 1.0;

   //--- Apply ML multiplier from signal combination stats (only if enabled)
   if(INPUT_ENABLE_ML && INPUT_ML_INFERENCE_ON && g_trainingDataCount >= INPUT_MIN_TRADES_FOR_ML)
   {
      double mlMultiplier = GetMLConfidenceMultiplier(combination);
      mlMultiplier = MathMin(mlMultiplier, g_adaptive.confMultiplierCap);
      mlApplied = true;
      appliedMLMultiplier = mlMultiplier;
      mlNonNeutral = (MathAbs(mlMultiplier - 1.0) > 0.00001);
      conf *= mlMultiplier;
   }

   //--- Apply fingerprint boost (only if enabled)
   if(INPUT_ENABLE_FINGERPRINT)
   {
      double fpBoost = GetFingerprintBoost(fpId, combination);
      conf += fpBoost; // Add boost (not multiply)
   }

   //--- Apply AI adjustment (only if enabled)
   if(INPUT_AI_MODE != AI_OFF && INPUT_AI_BLEND_ON && g_aiResponse.lastUpdate > 0)
   {
      double aiAdjustment = (g_aiResponse.confidenceScore - 50.0) * INPUT_AI_WEIGHT;
      conf += aiAdjustment;

      // Risk alert penalty
      if(g_aiResponse.riskAlert)
         conf -= 10.0;
   }

   //--- Apply Markov streak adjustment (only if enabled)
   if(INPUT_ENABLE_MARKOV && INPUT_MARKOV_INFERENCE_ON)
   {
      double markovAdj = GetMarkovConfidenceAdjustment();
      conf += markovAdj;
   }

   //--- Priority 1: High ADX risk mode (optional)
   if(INPUT_LOT_HIGH_ADX_BOOST_ON && INPUT_ENABLE_HIGH_ADX_RISK_MODE)
   {
      double adxNow[];
      if(CopyBuffer(g_hADX_M1, 0, 0, 1, adxNow) == 1 && adxNow[0] >= INPUT_HIGH_ADX_THRESHOLD)
         conf += INPUT_HIGH_ADX_CONFIDENCE_BOOST;
   }

   //--- Priority 2: Combination-adaptive confidence (optional)
   bool allowComboAdaptive = (!mlNonNeutral);
   if(INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_INFERENCE_ON && allowComboAdaptive)
   {
      string canonicalSubsets[];
      int subsetCount = BuildCanonicalComboSubsets(combination, INPUT_MIN_SIGNALS, canonicalSubsets);
      double avgRank = 0.0;
      int rankCount = 0;
      for(int s = 0; s < subsetCount; s++)
      {
         for(int i = 0; i < g_combinationStatsCount; i++)
         {
            if(g_combinationStats[i].combination == canonicalSubsets[s] && g_combinationStats[i].totalTrades >= INPUT_COMBO_MIN_TRADES)
            {
               avgRank += g_combinationStats[i].rankScore;
               rankCount++;
               break;
            }
         }
      }
      if(rankCount > 0)
      {
         avgRank /= rankCount;
         double edge = (avgRank - 50.0) / 50.0; // -1..+1
         double comboMultiplier = (1.0 + edge * INPUT_COMBO_CONFIDENCE_WEIGHT);
         conf *= comboMultiplier;
         comboApplied = true;
         appliedComboMultiplier = comboMultiplier;
         comboNonNeutral = (MathAbs(comboMultiplier - 1.0) > 0.00001);
      }
   }

   double consecBoostApplied = 0.0;
   if(INPUT_ENABLE_CONSEC_WIN_CONF_BOOST && g_consecutiveWins >= INPUT_CONSEC_WIN_CONF_TRIGGER)
   {
      int boostWins = g_consecutiveWins - INPUT_CONSEC_WIN_CONF_TRIGGER + 1;
      consecBoostApplied = boostWins * INPUT_CONSEC_WIN_CONF_BOOST_PER_WIN;
      consecBoostApplied = MathMin(consecBoostApplied, INPUT_CONSEC_WIN_CONF_BOOST_CAP);
      if(INPUT_ENABLE_CONSEC_WIN_CONF_DECAY && g_consecWinBoostTrades >= INPUT_CONSEC_WIN_CONF_DECAY_AFTER_TRADES)
         consecBoostApplied *= 0.5;
      conf += consecBoostApplied;
   }

   double treeAdj = GetTreeConfidenceAdjustment(combination);
   conf += treeAdj;

   if(INPUT_ENABLE_LOGGING)
   {
      Print("CONF DEBUG: combo=", combination,
            " | raw=", DoubleToString(50.0 + trendComponent + momentumComponent + srComponent + (volComponent * 0.5) + timeComponent + divergenceComponent + (extraSignals > 0 ? extraSignals * 5.0 : 0.0), 2),
            " | trend=", DoubleToString(trendComponent, 2),
            " momentum=", DoubleToString(momentumComponent, 2),
            " sr=", DoubleToString(srComponent, 2),
            " vol=", DoubleToString(volComponent * 0.5, 2),
            " time=", DoubleToString(timeComponent, 2),
            " div=", DoubleToString(divergenceComponent, 2),
            " extraSig=", (extraSignals > 0 ? IntegerToString(extraSignals) : "0"),
            " | mlApplied=", (mlApplied ? "true" : "false"),
            " mlMul=", DoubleToString(appliedMLMultiplier, 3),
            " mlNon1=", (mlNonNeutral ? "true" : "false"),
            " | comboAllowed=", (allowComboAdaptive ? "true" : "false"),
            " comboApplied=", (comboApplied ? "true" : "false"),
            " comboMul=", DoubleToString(appliedComboMultiplier, 3),
            " comboNon1=", (comboNonNeutral ? "true" : "false"),
            " treeAdj=", DoubleToString(treeAdj, 3),
            " consecBoost=", DoubleToString(consecBoostApplied, 2));
   }

   //--- Canonical threat penalty budget: keep a single confidence penalty path
   if(threat > 60)
      conf -= (threat - 60) * 0.12;

   //--- Final clamp
   if(conf < 0) conf = 0;
   if(conf > 100) conf = 100;

   return conf;
}
//+------------------------------------------------------------------+
// (trend, momentum, SR, volatility, time, divergence components - unchanged)
//+------------------------------------------------------------------+
double CalculateTrendStrengthComponent(int direction)
{
   double component = 0;
   double emaFast[], emaSlow[], emaTrend[];
   double adx[], plusDI[], minusDI[];

   if(CopyBuffer(g_hEmaFast_M1, 0, 0, 10, emaFast) < 10) return 5.0; // FIXED: Default positive
   if(CopyBuffer(g_hEmaSlow_M1, 0, 0, 10, emaSlow) < 10) return 5.0;
   if(CopyBuffer(g_hEmaTrend_M1, 0, 0, 5, emaTrend) < 5) return 5.0;
   if(CopyBuffer(g_hADX_M1, 0, 0, 3, adx) < 3) return 5.0;
   if(CopyBuffer(g_hADX_M1, 1, 0, 3, plusDI) < 3) return 5.0;
   if(CopyBuffer(g_hADX_M1, 2, 0, 3, minusDI) < 3) return 5.0;

   double currentPrice = iClose(_Symbol, PERIOD_M1, 1);

   //--- EMA Alignment (+0 to +15)
   if(direction == 1) // BUY
   {
      if(currentPrice > emaFast[1] && emaFast[1] > emaSlow[1] && emaSlow[1] > emaTrend[1])
         component += 15.0;
      else if(currentPrice > emaFast[1] && emaFast[1] > emaSlow[1])
         component += 10.0; // FIXED: Increased from 8
      else if(emaFast[1] > emaSlow[1])
         component += 5.0; // FIXED: Added partial credit
   }
   else // SELL
   {
      if(currentPrice < emaFast[1] && emaFast[1] < emaSlow[1] && emaSlow[1] < emaTrend[1])
         component += 15.0;
      else if(currentPrice < emaFast[1] && emaFast[1] < emaSlow[1])
         component += 10.0;
      else if(emaFast[1] < emaSlow[1])
         component += 5.0;
   }

   //--- EMA Slope (+0 to +8), normalized by ATR to stay symbol-agnostic.
   // Convert 4-bar EMA change into per-bar slope, then compare against ATR fractions.
   double atrSlope[];
   if(CopyBuffer(g_hATR_M1, 0, 0, 2, atrSlope) < 2 || atrSlope[1] <= 0)
      return MathMin(component + 2.0, 35.0); // minimal credit if ATR unavailable

   double emaSlopePerBar = (emaFast[1] - emaFast[5]) / 4.0;
   double normalizedSlope = emaSlopePerBar / atrSlope[1];
   const double EMA_SLOPE_WEAK_THRESHOLD = INPUT_EMA_SLOPE_ATR_WEAK;   // e.g. 0.5% ATR/bar
   const double EMA_SLOPE_STRONG_THRESHOLD = INPUT_EMA_SLOPE_ATR_STRONG; // e.g. 1.0% ATR/bar

   if(direction == 1 && normalizedSlope > EMA_SLOPE_STRONG_THRESHOLD) component += 8.0;
   else if(direction == 1 && normalizedSlope > EMA_SLOPE_WEAK_THRESHOLD) component += 4.0;
   else if(direction == 1 && normalizedSlope > 0) component += 2.0;
   else if(direction == -1 && normalizedSlope < -EMA_SLOPE_STRONG_THRESHOLD) component += 8.0;
   else if(direction == -1 && normalizedSlope < -EMA_SLOPE_WEAK_THRESHOLD) component += 4.0;
   else if(direction == -1 && normalizedSlope < 0) component += 2.0;

   //--- ADX Confirmation (+0 to +12) - FIXED: Reduced penalties
   if(adx[1] >= 40) component += 12.0;
   else if(adx[1] >= 25) component += 8.0;
   else if(adx[1] >= 20) component += 4.0; // FIXED: Added points
   else if(adx[1] >= 15) component += 2.0; // FIXED: Added points
   // FIXED: Removed negative penalty for low ADX

   return MathMin(component, 35.0); // Cap at +35
}
//+------------------------------------------------------------------+
double CalculateMomentumComponent(int direction)
{
   double component = 0;
   double rsi[], macdMain[], macdSignal[], stochK[], stochD[];

   if(CopyBuffer(g_hRSI_M1, 0, 0, 5, rsi) < 5) return 5.0; // FIXED: Default positive
   if(CopyBuffer(g_hMACD_M1, 0, 0, 5, macdMain) < 5) return 5.0;
   if(CopyBuffer(g_hMACD_M1, 1, 0, 5, macdSignal) < 5) return 5.0;
   if(CopyBuffer(g_hStoch_M1, 0, 0, 3, stochK) < 3) return 5.0;
   if(CopyBuffer(g_hStoch_M1, 1, 0, 3, stochD) < 3) return 5.0;

   //--- RSI Analysis (-4 to +6) FIXED: Reduced penalties
   if(direction == 1) // BUY
   {
      if(rsi[1] < 30) component += 6.0;       // Oversold bounce
      else if(rsi[1] < 40) component += 4.0;  // Mild oversold
      else if(rsi[1] < 50) component += 2.0; // FIXED: Added
      else if(rsi[1] > 70) component -= 4.0; // FIXED: Reduced from -8
      else if(rsi[1] > 60) component -= 1.0;  // FIXED: Reduced from -3
   }
   else // SELL
   {
      if(rsi[1] > 70) component += 6.0;       // Overbought reversal
      else if(rsi[1] > 60) component += 4.0;
      else if(rsi[1] > 50) component += 2.0; // FIXED: Added
      else if(rsi[1] < 30) component -= 4.0;  // FIXED: Reduced
      else if(rsi[1] < 40) component -= 1.0;  // FIXED: Reduced
   }

   //--- MACD Analysis (-5 to +10) FIXED: Reduced penalties
   if(direction == 1)
   {
      if(macdMain[1] > macdSignal[1] && macdMain[1] > 0) component += 10.0;
      else if(macdMain[1] > macdSignal[1]) component += 5.0; // FIXED: Increased from 3
      else if(macdMain[1] < macdSignal[1] && macdMain[1] < 0) component -= 5.0; // FIXED: Reduced from -10
   }
   else
   {
      if(macdMain[1] < macdSignal[1] && macdMain[1] < 0) component += 10.0;
      else if(macdMain[1] < macdSignal[1]) component += 5.0;
      else if(macdMain[1] > macdSignal[1] && macdMain[1] > 0) component -= 5.0;
   }

   //--- Histogram acceleration (+0 to +5) - FIXED: Removed penalty
   double histNow = macdMain[1] - macdSignal[1];
   double histPrev = macdMain[2] - macdSignal[2];
   if((direction == 1 && histNow > histPrev) ||
      (direction == -1 && histNow < histPrev))
      component += 5.0;
   // FIXED: Removed else penalty

   //--- Stochastic (-3 to +6) FIXED: Reduced penalties
   if(direction == 1 && stochK[1] < 20) component += 6.0;
   else if(direction == 1 && stochK[1] > 80) component -= 3.0; // FIXED: Reduced from -6
   else if(direction == -1 && stochK[1] > 80) component += 6.0;
   else if(direction == -1 && stochK[1] < 20) component -= 3.0;

   return MathMax(MathMin(component, 21.0), -5.0); // FIXED: Limit downside
}
//+------------------------------------------------------------------+
double CalculateSRComponent(int direction)
{
   double component = 0;

   // Calculate recent high/low
   double highestHigh = iHigh(_Symbol, PERIOD_M1, 2);
   double lowestLow = iLow(_Symbol, PERIOD_M1, 2);
   for(int i = 3; i <= INPUT_BREAKOUT_LOOKBACK + 1; i++)
   {
      double h = iHigh(_Symbol, PERIOD_M1, i);
      double l = iLow(_Symbol, PERIOD_M1, i);
      if(h > highestHigh) highestHigh = h;
      if(l < lowestLow) lowestLow = l;
   }

   double currentPrice = iClose(_Symbol, PERIOD_M1, 1);
   double range = highestHigh - lowestLow;
   if(range <= 0) return 0;

   double distToResistance = highestHigh - currentPrice;
   double distToSupport = currentPrice - lowestLow;
   double nearThreshold = range * 0.1; // Within 10% of level

   if(direction == 1) // BUY
   {
      if(distToSupport < nearThreshold)
         component += 8.0; // Bounce from support
      else if(distToResistance < nearThreshold)
         component -= 4.0; // FIXED: Reduced from -8
   }
   else // SELL
   {
      if(distToResistance < nearThreshold)
         component += 8.0;
      else if(distToSupport < nearThreshold)
         component -= 4.0; // FIXED: Reduced from -8
   }

   return component;
}
//+------------------------------------------------------------------+
double CalculateVolatilityComponent()
{
   double component = 0;
   double volRatio = CalculateVolatilityRatio();

   // FIXED: Reduced all penalties
   if(volRatio < 0.7) component -= 4.0; // FIXED: Reduced from -8
   else if(volRatio >= 0.7 && volRatio < 1.2) component += 2.0; // FIXED: Added bonus
   else if(volRatio >= 1.2 && volRatio < 1.5) component -= 1.0; // FIXED: Reduced from -3
   else if(volRatio >= 1.5) component -= 5.0; // FIXED: Reduced from -10

   // (BB width check removed - was causing too many rejections)

   return MathMax(component, -5.0); // FIXED: Limit downside
}
//+------------------------------------------------------------------+
double CalculateTimeComponent()
{
   double component = 0;
   int session = GetCurrentSession();

   // Session bonus
   if(session == 1) component += 8.0;      // London
   else if(session == 2) component += 5.0; // NY
   else if(session == 0) component += 3.0; // Asian - FIXED: Increased from 2
   else component += 1.0; // FIXED: Added default instead of 0

   // Day of week - FIXED: Reduced penalties
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);

   if(dt.day_of_week == 5)      component -= 1.0; // FIXED: Reduced from -3
   else if(dt.day_of_week >= 1 && dt.day_of_week <= 4)
      component += 2.0; // Mon-Thu

   // (micro?timing penalties removed)

   return MathMin(component, 13.0);
}
//+------------------------------------------------------------------+
double CalculateDivergenceComponent(int direction)
{
   double component = 0;

   //--- RSI Divergence detection (simplified)
   double rsi[];
   if(CopyBuffer(g_hRSI_M1, 0, 0, 20, rsi) < 20) return 0;

   double high1 = iHigh(_Symbol, PERIOD_M1, 1);
   double high10 = iHigh(_Symbol, PERIOD_M1, 10);
   double low1 = iLow(_Symbol, PERIOD_M1, 1);
   double low10 = iLow(_Symbol, PERIOD_M1, 10);

   // Bearish divergence: price higher high, RSI lower high
   if(high1 > high10 && rsi[1] < rsi[10])
   {
      if(direction == -1) component += 8.0; // Good for sells
      // FIXED: Removed penalty for buys
   }
   // Bullish divergence: price lower low, RSI higher low
   if(low1 < low10 && rsi[1] > rsi[10])
   {
      if(direction == 1) component += 8.0; // Good for buys
      // FIXED: Removed penalty for sells
   }

   //--- Pin bar pattern detection
   double open1 = iOpen(_Symbol, PERIOD_M1, 1);
   double close1 = iClose(_Symbol, PERIOD_M1, 1);
   double range1 = high1 - low1;
   double body1 = MathAbs(close1 - open1);

   if(range1 > 0 && body1 < range1 * 0.3) // Pin bar (small body, long wicks)
   {
      double upperWick = high1 - MathMax(open1, close1);
      double lowerWick = MathMin(open1, close1) - low1;

      // Bullish pin bar (long lower wick)
      if(lowerWick > upperWick * 2 && direction == 1)
         component += 6.0;
      // Bearish pin bar (long upper wick)
      else if(upperWick > lowerWick * 2 && direction == -1)
         component += 6.0;
   }

   return MathMin(component, 14.0);
}
//+------------------------------------------------------------------+
//| SECTION 13: 8-SIGNAL DETECTION - FIXED!                          |
//+------------------------------------------------------------------+
bool DetectSignals(SignalResult &signals)
{
   ZeroMemory(signals);

   //--- Get indicator values
   double emaFast[], emaSlow[];
   double rsi[];
   double stochK[], stochD[];
   double macdMain[], macdSignal[];
   double wpr[];
   double atr[];
   double bbUpper[], bbLower[], bbMiddle[];
   double volume[];

   if(CopyBuffer(g_hEmaFast_M1, 0, 0, 5, emaFast) < 5) return false;
   if(CopyBuffer(g_hEmaSlow_M1, 0, 0, 5, emaSlow) < 5) return false;
   if(CopyBuffer(g_hRSI_M1, 0, 0, 5, rsi) < 5) return false;
   if(CopyBuffer(g_hStoch_M1, 0, 0, 5, stochK) < 5) return false;
   if(CopyBuffer(g_hStoch_M1, 1, 0, 5, stochD) < 5) return false;
   if(CopyBuffer(g_hMACD_M1, 0, 0, 5, macdMain) < 5) return false;
   if(CopyBuffer(g_hMACD_M1, 1, 0, 5, macdSignal) < 5) return false;
   if(CopyBuffer(g_hWPR_M1, 0, 0, 5, wpr) < 5) return false;
   if(CopyBuffer(g_hATR_M1, 0, 0, 3, atr) < 3) return false;
   if(CopyBuffer(g_hBB_M1, 1, 0, 3, bbUpper) < 3) return false;
   if(CopyBuffer(g_hBB_M1, 2, 0, 3, bbLower) < 3) return false;
   if(CopyBuffer(g_hBB_M1, 0, 0, 3, bbMiddle) < 3) return false;
   if(CopyBuffer(g_hVolume_M1, 0, 0, 5, volume) < 5) return false;

   int b = 1;   // completed bar index
   int b2 = 2;  // previous bar
   int b3 = 3;  // 2?bars?ago (used for look?back crossovers)

   //--- Get OHLC of required bars
   double open1 = iOpen(_Symbol, PERIOD_M1, 1);
   double close1 = iClose(_Symbol, PERIOD_M1, 1);
   double high1 = iHigh(_Symbol, PERIOD_M1, 1);
   double low1 = iLow(_Symbol, PERIOD_M1, 1);
   double open2 = iOpen(_Symbol, PERIOD_M1, 2);
   double close2 = iClose(_Symbol, PERIOD_M1, 2);
   double high2 = iHigh(_Symbol, PERIOD_M1, 2);
   double low2 = iLow(_Symbol, PERIOD_M1, 2);

   //--- Signal 1: EMA Crossover (current or last 2 bars)
   if(emaFast[b] > emaSlow[b] && emaFast[b2] <= emaSlow[b2])
   {
      signals.emaSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(emaFast[b] < emaSlow[b] && emaFast[b2] >= emaSlow[b2])
   {
      signals.emaSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // Look?back crossovers (now also accept within the last 3 bars)
   else if(emaFast[b] > emaSlow[b] && (emaFast[b2] <= emaSlow[b2] || emaFast[b3] <= emaSlow[b3]))
   {
      signals.emaSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(emaFast[b] < emaSlow[b] && (emaFast[b2] >= emaSlow[b2] || emaFast[b3] >= emaSlow[b3]))
   {
      signals.emaSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // V7.2 FIX (BUG 10): REMOVED fallback EMA alignment - it used to fire every bar.

   //--- Signal 2: RSI Oversold/Overbought (tightened zones)
   if(rsi[b] < 35) // FIXED: Changed from 30
   {
      signals.rsiSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(rsi[b] > 65) // FIXED: Changed from 70
   {
      signals.rsiSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // V7.2 FIX (BUG 10): Removed mid?range RSI signals.

   //--- Signal 3: Stochastic Crossover in OS/OB zones (tightened)
   if(stochK[b] < 25 && stochK[b] > stochD[b]) // FIXED: Changed from 20
   {
      signals.stochSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(stochK[b] > 75 && stochK[b] < stochD[b]) // FIXED: Changed from 80
   {
      signals.stochSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // V7.2 FIX (BUG 10): Removed mid?range stochastic signals.

   //--- Signal 4: Engulfing Pattern (plus strong candle)
   bool bullEngulfing = (close2 < open2) && (close1 > open1) && (close1 > open2) && (open1 < close2);
   bool bearEngulfing = (close2 > open2) && (close1 < open1) && (close1 < open2) && (open1 > close2);

   if(bullEngulfing)
   {
      signals.engulfingSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(bearEngulfing)
   {
      signals.engulfingSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // Add strong single?candle check
   else if(close1 > open1 && (close1 - open1) > (high1 - low1) * 0.6)
   {
      signals.engulfingSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(close1 < open1 && (open1 - close1) > (high1 - low1) * 0.6)
   {
      signals.engulfingSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }

   //--- Signal 5: Breakout (excluding the signal bar itself)
   double highestHigh = iHigh(_Symbol, PERIOD_M1, 2);
   double lowestLow   = iLow(_Symbol, PERIOD_M1, 2);
   for(int i = 3; i <= INPUT_BREAKOUT_LOOKBACK + 1; i++)
   {
      double h = iHigh(_Symbol, PERIOD_M1, i);
      double l = iLow(_Symbol, PERIOD_M1, i);
      if(h > highestHigh) highestHigh = h;
      if(l < lowestLow)   lowestLow = l;
   }

   if(close1 > highestHigh)
   {
      signals.breakoutSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(close1 < lowestLow)
   {
      signals.breakoutSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // Near?breakout addition
   else if(close1 > highestHigh * 0.998)
   {
      signals.breakoutSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(close1 < lowestLow * 1.002)
   {
      signals.breakoutSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }

   //--- Signal 6: Volume Spike (threshold lowered)
   double avgVolume = 0;
   double volArr[];
   if(CopyBuffer(g_hVolume_M1, 0, 1, INPUT_VOLUME_AVG_PERIOD, volArr) == INPUT_VOLUME_AVG_PERIOD)
   {
      for(int v = 0; v < INPUT_VOLUME_AVG_PERIOD; v++)
         avgVolume += volArr[v];
      avgVolume /= INPUT_VOLUME_AVG_PERIOD;

      if(volume[b] > avgVolume * 2.0) // FIXED: Changed from 2.0
      {
         signals.volumeSignal = true;
         if(close1 > open1)
         {
            signals.bullVotes++;
            signals.totalSignals++;
         }
         else if(close1 < open1)
         {
            signals.bearVotes++;
            signals.totalSignals++;
         }
         // V7.2 FIX (BUG 11): REMOVED neutral volume signal (no direction ? no vote)
      }
   }

   //--- Signal 7: MACD Crossover (no fallback)
   if(macdMain[b] > macdSignal[b] && macdMain[b2] <= macdSignal[b2])
   {
      signals.macdSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(macdMain[b] < macdSignal[b] && macdMain[b2] >= macdSignal[b2])
   {
      signals.macdSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // V7.2 FIX (BUG 10): Removed fallback alignment.

   //--- Signal 8: Williams %R (tightened zones)
   if(wpr[b] < -75) // FIXED: Changed from -80
   {
      signals.wprSignal = true;
      signals.bullVotes++;
      signals.totalSignals++;
   }
   else if(wpr[b] > -25) // FIXED: Changed from -20
   {
      signals.wprSignal = true;
      signals.bearVotes++;
      signals.totalSignals++;
   }
   // V7.2 FIX (BUG 10): Removed mid?range WPR signals.

   //--- Generate combination string
   signals.combinationString = GenerateSignalCombinationString(signals);

   return true;
}
//+------------------------------------------------------------------+
string GenerateSignalCombinationString(const SignalResult &signals)
{
   string combo = "";

   if(signals.emaSignal)      combo += "EMA_";
   if(signals.rsiSignal)      combo += "RSI_";
   if(signals.stochSignal)    combo += "STOCH_";
   if(signals.engulfingSignal)combo += "ENGULF_";
   if(signals.breakoutSignal)combo += "BREAK_";
   if(signals.volumeSignal)   combo += "VOL_";
   if(signals.macdSignal)    combo += "MACD_";
   if(signals.wprSignal)     combo += "WPR_";

   if(StringLen(combo) > 0)
      combo = StringSubstr(combo, 0, StringLen(combo) - 1); // Remove trailing underscore

   return combo;
}
//+------------------------------------------------------------------+
//| SECTION 14: MTF ALIGNMENT (Weighted by timeframe)                |
//+------------------------------------------------------------------+
int CalculateMTFAlignment(int direction)
{
   int score = 0;
 g_lastMtfAlignmentHadReadFailure = false;
   //--- M5 (weight: 1)
   double m5Fast[], m5Slow[];
   
      int m5FastRead = CopyBuffer(g_hEmaFast_M5, 0, 0, 1, m5Fast);
   int m5SlowRead = CopyBuffer(g_hEmaSlow_M5, 0, 0, 1, m5Slow);
   if(m5FastRead == 1 && m5SlowRead == 1)
   {
      if((direction == 1 && m5Fast[0] > m5Slow[0]) ||
         (direction == -1 && m5Fast[0] < m5Slow[0]))
         score += 1;
   }

 else
   {
      g_lastMtfAlignmentHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: CalculateMTFAlignment M5 fastRead=" + IntegerToString(m5FastRead) +
                             " slowRead=" + IntegerToString(m5SlowRead));
   }

   // Closed candles for higher timeframes prevent intrabar flips (start_pos=1).
   //--- H1 (weight: 2)
   double h1Fast[], h1Slow[];
   int h1FastRead = CopyBuffer(g_hEmaFast_H1, 0, 1, 1, h1Fast);
   int h1SlowRead = CopyBuffer(g_hEmaSlow_H1, 0, 1, 1, h1Slow);
   if(h1FastRead == 1 && h1SlowRead == 1)
   {
      if((direction == 1 && h1Fast[0] > h1Slow[0]) ||
         (direction == -1 && h1Fast[0] < h1Slow[0]))
         score += 2;
   }
 else
   {
      g_lastMtfAlignmentHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: CalculateMTFAlignment H1 fastRead=" + IntegerToString(h1FastRead) +
                             " slowRead=" + IntegerToString(h1SlowRead));
   }
   //--- H4 (weight: 3)
   double h4Fast[], h4Slow[];
  int h4FastRead = CopyBuffer(g_hEmaFast_H4, 0, 1, 1, h4Fast);
   int h4SlowRead = CopyBuffer(g_hEmaSlow_H4, 0, 1, 1, h4Slow);
   if(h4FastRead == 1 && h4SlowRead == 1)
   {
      if((direction == 1 && h4Fast[0] > h4Slow[0]) ||
         (direction == -1 && h4Fast[0] < h4Slow[0]))
         score += 3;
   }
else
   {
      g_lastMtfAlignmentHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: CalculateMTFAlignment H4 fastRead=" + IntegerToString(h4FastRead) +
                             " slowRead=" + IntegerToString(h4SlowRead));
   }
   //--- D1 (weight: 4)
   double d1Fast[], d1Slow[];
  int d1FastRead = CopyBuffer(g_hEmaFast_D1, 0, 1, 1, d1Fast);
   int d1SlowRead = CopyBuffer(g_hEmaSlow_D1, 0, 1, 1, d1Slow);
   if(d1FastRead == 1 && d1SlowRead == 1)
   
   {
      if((direction == 1 && d1Fast[0] > d1Slow[0]) ||
         (direction == -1 && d1Fast[0] < d1Slow[0]))
         score += 4;
   }
else
   {
      g_lastMtfAlignmentHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: CalculateMTFAlignment D1 fastRead=" + IntegerToString(d1FastRead) +
                             " slowRead=" + IntegerToString(d1SlowRead));
   }
   
   return score;
}
//+------------------------------------------------------------------+
int GetTimeframeDirectionConsensus(int &agreeingFrames)
{
   agreeingFrames = 0;
   int bullishFrames = 0;
   int bearishFrames = 0;

 g_lastMtfConsensusHadReadFailure = false;
 
   // Consensus is intentionally based on higher-timeframe structure only.
   // Closed candles (start_pos=1) are used on H1/H4/D1 to avoid intrabar flips.
   double h1Fast[], h1Slow[];
int h1FastRead = CopyBuffer(g_hEmaFast_H1, 0, 1, 1, h1Fast);
   int h1SlowRead = CopyBuffer(g_hEmaSlow_H1, 0, 1, 1, h1Slow);
   if(h1FastRead == 1 && h1SlowRead == 1)
   {
      if(h1Fast[0] > h1Slow[0]) bullishFrames++;
      else if(h1Fast[0] < h1Slow[0]) bearishFrames++;
   }
 else
   {
      g_lastMtfConsensusHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: GetTimeframeDirectionConsensus H1 fastRead=" + IntegerToString(h1FastRead) +
                             " slowRead=" + IntegerToString(h1SlowRead));
   }
   
   double h4Fast[], h4Slow[];
    int h4FastRead = CopyBuffer(g_hEmaFast_H4, 0, 1, 1, h4Fast);
   int h4SlowRead = CopyBuffer(g_hEmaSlow_H4, 0, 1, 1, h4Slow);
   if(h4FastRead == 1 && h4SlowRead == 1)
   {
      if(h4Fast[0] > h4Slow[0]) bullishFrames++;
      else if(h4Fast[0] < h4Slow[0]) bearishFrames++;
   }
else
   {
      g_lastMtfConsensusHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: GetTimeframeDirectionConsensus H4 fastRead=" + IntegerToString(h4FastRead) +
                             " slowRead=" + IntegerToString(h4SlowRead));
   }
   
   double d1Fast[], d1Slow[];
    int d1FastRead = CopyBuffer(g_hEmaFast_D1, 0, 1, 1, d1Fast);
   int d1SlowRead = CopyBuffer(g_hEmaSlow_D1, 0, 1, 1, d1Slow);
   if(d1FastRead == 1 && d1SlowRead == 1)
   {
      if(d1Fast[0] > d1Slow[0]) bullishFrames++;
      else if(d1Fast[0] < d1Slow[0]) bearishFrames++;
   }
 else
   {
      g_lastMtfConsensusHadReadFailure = true;
      g_mtfReadFailureThisTick = true;
      g_gateDiagnostics.mtfDataReadRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF DATA READ FAILED: GetTimeframeDirectionConsensus D1 fastRead=" + IntegerToString(d1FastRead) +
                             " slowRead=" + IntegerToString(d1SlowRead));
   }
   
   if(bullishFrames >= 2)
   {
      agreeingFrames = bullishFrames;
      return 1;
   }
   if(bearishFrames >= 2)
   {
      agreeingFrames = bearishFrames;
      return -1;
   }

   agreeingFrames = MathMax(bullishFrames, bearishFrames);
   return 0;
}
//+------------------------------------------------------------------+
//| SECTION 15: Q-LEARNING SYSTEM (Part 4 of Strategy)               |
//+------------------------------------------------------------------+
int DetermineRLState(double confidence, double threat, int positions,
                     double drawdown, bool winStreak)
{
   // State encoding: 3x3x3x2x2 = 108 states
   // Confidence: LOW(0), MEDIUM(1), HIGH(2)
   // Threat: SAFE(0), MODERATE(1), DANGEROUS(2)
   // Positions: FEW(0), SEVERAL(1), MANY(2)
   // Drawdown: HEALTHY(0), STRESSED(1)
   // Performance: LOSING(0), WINNING(1)

   int confBucket = 0;
   if(confidence >= 75) confBucket = 2;      // HIGH
   else if(confidence >= 55) confBucket = 1; // MEDIUM
   // else LOW = 0

   int threatBucket = 0;
   if(threat > 70) threatBucket = 2;      // DANGEROUS
   else if(threat > 40) threatBucket = 1; // MODERATE
   // else SAFE = 0

   int posBucket = 0;
   if(positions >= 4) posBucket = 2;      // MANY
   else if(positions >= 2) posBucket = 1; // SEVERAL
   // else FEW = 0

   int ddBucket = (drawdown >= 2.0) ? 1 : 0;
   int perfBucket = winStreak ? 1 : 0;

   // Calculate state index
   int state = confBucket * 36 + threatBucket * 12 + posBucket * 4 + ddBucket * 2 + perfBucket;
   return MathMin(state, Q_TABLE_STATES - 1);
}
//+------------------------------------------------------------------+
ENUM_RL_ACTION GetRLAction(int state)
{
   if(!INPUT_ENABLE_RL || g_rlTradesCompleted < INPUT_RL_MIN_TRADES)
      return RL_FULL_SIZE; // Default before learning

   if(!g_rngSeeded)
   {
      int seed = (int)(TimeLocal() ^ (datetime)GetTickCount());
      MathSrand(seed);
      g_rngSeeded = true;
      Print("RNG SEEDED LATE: applied defensive seed in GetRLAction | seed=", seed);
   }

   // Epsilon?greedy: explore with probability epsilon
   double rand = (double)MathRand() / 32767.0;

   if(rand < INPUT_RL_EPSILON)
   {
      // Explore: random action
      int randAction = MathRand() % Q_TABLE_ACTIONS;
      return (ENUM_RL_ACTION)randAction;
   }
   else
   {
      // Exploit: best known action
      double maxQ = g_qTable[state][0];
      int bestAction = 0;

      for(int a = 1; a < Q_TABLE_ACTIONS; a++)
      {
         if(g_qTable[state][a] > maxQ)
         {
            maxQ = g_qTable[state][a];
            bestAction = a;
         }
      }
      return (ENUM_RL_ACTION)bestAction;
   }
}
//+------------------------------------------------------------------+
double GetCombinationStrengthSnapshot(const string &combination)
{
   string canonicalSubsets[];
   int subsetCount = BuildCanonicalComboSubsets(combination, INPUT_MIN_SIGNALS, canonicalSubsets);
   if(subsetCount <= 0) return 50.0;

   double sumStrength = 0.0;
   int matches = 0;
   for(int s = 0; s < subsetCount; s++)
   {
      for(int i = 0; i < g_combinationStatsCount; i++)
      {
         if(g_combinationStats[i].combination != canonicalSubsets[s]) continue;
         sumStrength += g_combinationStats[i].strengthScore;
         matches++;
         break;
      }
   }
   if(matches <= 0) return 50.0;
   return sumStrength / matches;
}
//+------------------------------------------------------------------+
double ResolveNextStateConfidence(const RLStateAction &rec)
{
   if(MathIsValidNumber(rec.confidenceSnapshot) && rec.confidenceSnapshot >= 0.0 && rec.confidenceSnapshot <= 100.0)
      return rec.confidenceSnapshot;

   // Deterministic fallback: combine MTF score and combo strength snapshot.
   double mtfComponent = 50.0;
   if(rec.mtfScoreSnapshot >= -4 && rec.mtfScoreSnapshot <= 4)
      mtfComponent = 50.0 + rec.mtfScoreSnapshot * 7.5;

   double comboComponent = 50.0;
   if(MathIsValidNumber(rec.comboStrengthSnapshot))
      comboComponent = MathMax(0.0, MathMin(100.0, rec.comboStrengthSnapshot));

   return MathMax(0.0, MathMin(100.0, (mtfComponent * 0.4) + (comboComponent * 0.6)));
}
//+------------------------------------------------------------------+
void RecordStateAction(int state, ENUM_RL_ACTION action, ulong orderTicket, ulong positionId,
                       double entryPrice, double slDistance, double lot, double tickValue,
                       double confidenceSnapshot, int mtfScoreSnapshot, double comboStrengthSnapshot)
{
   const int pendingCap = MathMax(0, INPUT_RL_PENDING_HARD_CAP);

   if(orderTicket == 0 && positionId == 0)
   {
      if(INPUT_ENABLE_LOGGING) Print("RL Pending skipped: missing order/position ticket");
      return;
   }

   datetime now = TimeCurrent();
   int maxAgeSec = INPUT_RL_PENDING_MAX_AGE_HOURS * 3600;
   if(maxAgeSec > 0 && g_pendingRLCount > 0)
   {
      int writeIdx = 0;
      int removed = 0;
      for(int i = 0; i < g_pendingRLCount; i++)
      {
         bool stale = ((now - g_pendingRL[i].timestamp) > maxAgeSec);
         if(stale)
         {
            removed++;
            continue;
         }

         if(writeIdx != i)
            g_pendingRL[writeIdx] = g_pendingRL[i];
         writeIdx++;
      }
      g_pendingRLCount = writeIdx;

      if(removed > 0 && INPUT_ENABLE_LOGGING)
         Print("RL Pending cleanup during record: removed stale entries=", removed,
               " | Remaining=", g_pendingRLCount);
   }

   if(g_pendingRLCount >= pendingCap)
   {
      if(INPUT_ENABLE_LOGGING)
         Print("RL WARNING: Pending RL buffer full (cap=", pendingCap,
               ") and no stale entries removable. Skipping new record for positionId=", positionId);
      return;
   }

   if(g_pendingRLCount >= ArraySize(g_pendingRL))
   {
      int targetSize = MathMin(pendingCap, g_pendingRLCount + 50);
      ArrayResize(g_pendingRL, targetSize);
   }

   if(g_pendingRLCount >= ArraySize(g_pendingRL))
   {
      if(INPUT_ENABLE_LOGGING)
         Print("RL WARNING: Pending RL buffer resize limited by cap. Skipping positionId=", positionId);
      return;
   }


   g_pendingRL[g_pendingRLCount].state = state;
   g_pendingRL[g_pendingRLCount].action = action;
   g_pendingRL[g_pendingRLCount].timestamp = TimeCurrent();
   g_pendingRL[g_pendingRLCount].orderTicket = orderTicket;
   g_pendingRL[g_pendingRLCount].positionTicket = positionId;
   g_pendingRL[g_pendingRLCount].entryPrice = entryPrice;
   g_pendingRL[g_pendingRLCount].slDistance = slDistance;
   g_pendingRL[g_pendingRLCount].lot = lot;
   g_pendingRL[g_pendingRLCount].tickValue = tickValue;
   g_pendingRL[g_pendingRLCount].confidenceSnapshot = confidenceSnapshot;
   g_pendingRL[g_pendingRLCount].mtfScoreSnapshot = mtfScoreSnapshot;
   g_pendingRL[g_pendingRLCount].comboStrengthSnapshot = comboStrengthSnapshot;
   g_pendingRLCount++;

   if(INPUT_ENABLE_LOGGING)
      Print("RL Pending: State=", state, " Action=", EnumToString(action),
                     " | OrderTicket=", orderTicket,
                     " | PositionId=", positionId);
}
//+------------------------------------------------------------------+



void RemovePendingRLByOrderTicket(ulong orderTicket)
{
   if(orderTicket == 0 || g_pendingRLCount <= 0) return;

   for(int i = g_pendingRLCount - 1; i >= 0; i--)
   {
      if(g_pendingRL[i].orderTicket != orderTicket || g_pendingRL[i].positionTicket != 0)
         continue;

      for(int j = i; j < g_pendingRLCount - 1; j++)
         g_pendingRL[j] = g_pendingRL[j + 1];
      g_pendingRLCount--;
   }
}
//+------------------------------------------------------------------+
void RemapPendingRLToPosition(ulong orderTicket, ulong positionId)
{
   if(orderTicket == 0 || positionId == 0) return;

   for(int i = 0; i < g_pendingRLCount; i++)
   {
      if(g_pendingRL[i].orderTicket != orderTicket)
         continue;

      g_pendingRL[i].positionTicket = positionId;
      g_pendingRL[i].timestamp = TimeCurrent();
      if(INPUT_ENABLE_LOGGING)
         Print("RL Pending remap: orderTicket=", orderTicket, " -> positionId=", positionId);
      return;
   }
}
//+------------------------------------------------------------------+
bool GetPendingRLRiskBasis(ulong positionId, double &entryPrice, double &slDistance, double &lot, double &tickValue)
{
   entryPrice = 0.0;
   slDistance = 0.0;
   lot = 0.0;
   tickValue = 0.0;

   for(int i = 0; i < g_pendingRLCount; i++)
   {
      if(g_pendingRL[i].positionTicket != positionId)
         continue;

      entryPrice = g_pendingRL[i].entryPrice;
      slDistance = g_pendingRL[i].slDistance;
      lot = g_pendingRL[i].lot;
      tickValue = g_pendingRL[i].tickValue;
      return true;
   }

   return false;
}

bool ComputeNormalizedRLReward(ulong positionId, double netProfit,
                               double &normalizedReward,
                               double &entryPrice, double &slDistance,
                               double &lot, double &tickValue,
                               double &riskBasis)
{
   normalizedReward = netProfit;
   riskBasis = 0.0;

   bool hasPendingBasis = GetPendingRLRiskBasis(positionId, entryPrice, slDistance, lot, tickValue);

   if(!hasPendingBasis)
   {
      if(PositionSelectByTicket(positionId))
      {
         entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         double sl = PositionGetDouble(POSITION_SL);
         slDistance = (sl > 0.0) ? MathAbs(entryPrice - sl) : 0.0;
         lot = PositionGetDouble(POSITION_VOLUME);
      }
      tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   }

   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   if(slDistance > 0.0 && lot > 0.0 && tickValue > 0.0 && tickSize > 0.0)
   {
      riskBasis = (slDistance / tickSize) * tickValue * lot;
      if(riskBasis > 0.0)
      {
         normalizedReward = netProfit / riskBasis;
         return true;
      }
   }

   normalizedReward = netProfit;
   return false;
}
void UpdateRLFromTrade(ulong positionId, double reward)
{
   bool matched = false;

   // Find pending RL record for this trade
   for(int i = 0; i < g_pendingRLCount; i++)
   {
      if(g_pendingRL[i].positionTicket == positionId)
      {
         matched = true;
         RLStateAction rec = g_pendingRL[i];
         int state = rec.state;
         ENUM_RL_ACTION action = rec.action;

         if(state < 0 || state >= Q_TABLE_STATES || action < 0 || action >= Q_TABLE_ACTIONS)
         {
            g_rlUnmatchedCloses++;
            if(INPUT_ENABLE_LOGGING)
               Print("RL WARNING: Dropping invalid state/action for positionId=", positionId,
                     " | state=", state, " | action=", (int)action);

            for(int j = i; j < g_pendingRLCount - 1; j++)
               g_pendingRL[j] = g_pendingRL[j + 1];
            g_pendingRLCount--;
            return;
         }

         // Bellman equation update
         double threat = CalculateMarketThreat();
         int positions = CountMainPositionsFromBroker();
         double dd = CalculateDrawdownPercent();
         double nextConfidence = ResolveNextStateConfidence(rec);
         int nextState = DetermineRLState(nextConfidence, threat, positions, dd, g_consecutiveWins > 0);
         double nextStateMaxQ = g_qTable[nextState][0];

         for(int a = 1; a < Q_TABLE_ACTIONS; a++)
            if(g_qTable[nextState][a] > nextStateMaxQ)
               nextStateMaxQ = g_qTable[nextState][a];

         double oldQ = g_qTable[state][action];
         double newQ = oldQ + INPUT_RL_ALPHA *
                       (reward + INPUT_RL_GAMMA * nextStateMaxQ - oldQ);
         g_qTable[state][action] = newQ;
         g_qVisits[state][action]++;

         // Remove processed entry (shift array)
         for(int j = i; j < g_pendingRLCount - 1; j++)
            g_pendingRL[j] = g_pendingRL[j + 1];
         g_pendingRLCount--;

         g_rlTradesCompleted++;
         g_rlMatchedUpdates++;

         if(INPUT_ENABLE_LOGGING)
            Print("RL Update: State=", state, " Action=", EnumToString(action),
                  " Reward=", reward, " OldQ=", oldQ, " NewQ=", newQ,
                  " | NextConf=", DoubleToString(nextConfidence, 2),
                  " | PositionId=", positionId,
                  " | PendingRemaining=", g_pendingRLCount);

         break;
      }
   }

   if(!matched)
   {
      g_rlUnmatchedCloses++;
      if(INPUT_ENABLE_LOGGING)
         Print("RL WARNING: No pending state/action found for closed positionId=", positionId,
               " | Reward=", reward,
               " | PendingCount=", g_pendingRLCount);
   }
}
//+------------------------------------------------------------------+
ENUM_RL_ACTION ApplyRLToDecision(double confidence, double threat,
                                  int positions, double drawdown)
{
   if(!INPUT_ENABLE_RL || g_rlTradesCompleted < INPUT_RL_MIN_TRADES)
      return RL_FULL_SIZE;

   int state = DetermineRLState(confidence, threat, positions, drawdown,
                              g_consecutiveWins >= 2);

   if(INPUT_ENABLE_META_POLICY)
   {
      int visits = 0;
      for(int a = 0; a < Q_TABLE_ACTIONS; a++)
         visits += MathMax(0, g_qVisits[state][a]);
      if(visits < INPUT_RL_MIN_STATE_VISITS)
         return RL_FULL_SIZE;
   }

   return GetRLAction(state);
}

void CleanupStalePendingRL()
{
   if(!INPUT_ENABLE_RL || g_pendingRLCount <= 0) return;

   datetime now = TimeCurrent();
   int writeIdx = 0;
   int removed = 0;
   int maxAgeSec = INPUT_RL_PENDING_MAX_AGE_HOURS * 3600;

   for(int i = 0; i < g_pendingRLCount; i++)
   {
      bool stale = (maxAgeSec > 0 && (now - g_pendingRL[i].timestamp) > maxAgeSec);
      bool unmatchedPendingGone = false;
      if(!stale && g_pendingRL[i].positionTicket == 0 && g_pendingRL[i].orderTicket > 0)
      {
         unmatchedPendingGone = !OrderSelect(g_pendingRL[i].orderTicket);
      }

      if(stale || unmatchedPendingGone)
      {
         removed++;
         continue;
      }

      if(writeIdx != i)
         g_pendingRL[writeIdx] = g_pendingRL[i];
      writeIdx++;
   }

   g_pendingRLCount = writeIdx;
   if(removed > 0 && INPUT_ENABLE_LOGGING)
      Print("RL CLEANUP: Removed stale/unmatched pending entries=", removed, " | Remaining=", g_pendingRLCount);
}

//+------------------------------------------------------------------+
//| SECTION 16: MARKOV CHAIN ANALYSIS (Part 4.2)                     |
//+------------------------------------------------------------------+
int GetMarkovRowTotal(const ENUM_MARKOV_STATE row)
{
   int rowTotal = 0;
   for(int to = 0; to < MARKOV_STATES; to++)
      rowTotal += MathMax(g_markovCounts[row][to], 0);
   return rowTotal;
}
//+------------------------------------------------------------------+
bool HasMarkovRowEvidence(const ENUM_MARKOV_STATE row, const int minimumRowSamples)
{
   return (GetMarkovRowTotal(row) >= minimumRowSamples);
}
//+------------------------------------------------------------------+
void RecomputeMarkovTransitionsFromCounts()
{
   const double laplaceAlpha = 1.0; // Dirichlet/Laplace smoothing for sparse rows.

   for(int from = 0; from < MARKOV_STATES; from++)
   {
      int rowTotal = GetMarkovRowTotal((ENUM_MARKOV_STATE)from);
      double denominator = rowTotal + (laplaceAlpha * MARKOV_STATES);

      for(int to = 0; to < MARKOV_STATES; to++)
      {
         double numerator = MathMax(g_markovCounts[from][to], 0) + laplaceAlpha;
         g_markovTransitions[from][to] = numerator / denominator;
      }
   }
}
//+------------------------------------------------------------------+
void UpdateMarkovTransition(ENUM_MARKOV_STATE fromState, ENUM_MARKOV_STATE toState)
{
   int lookback = MathMax(1, INPUT_MARKOV_LOOKBACK);
   if(ArraySize(g_markovQueue) != lookback)
   {
      ArrayResize(g_markovQueue, lookback);
      g_markovQueueCount = 0;
      g_markovQueueHead = 0;
      ArrayInitialize(g_markovCounts, 0);
      g_markovTradesRecorded = 0;
   }

   if(g_markovQueueCount >= lookback)
   {
      MarkovTransitionEvent oldEvent = g_markovQueue[g_markovQueueHead];
      g_markovCounts[oldEvent.fromState][oldEvent.toState] = MathMax(g_markovCounts[oldEvent.fromState][oldEvent.toState] - 1, 0);
      g_markovQueue[g_markovQueueHead].fromState = fromState;
      g_markovQueue[g_markovQueueHead].toState = toState;
      g_markovQueue[g_markovQueueHead].observedAt = TimeCurrent();
      g_markovQueueHead = (g_markovQueueHead + 1) % lookback;
   }
   else
   {
      int idx = (g_markovQueueHead + g_markovQueueCount) % lookback;
      g_markovQueue[idx].fromState = fromState;
      g_markovQueue[idx].toState = toState;
      g_markovQueue[idx].observedAt = TimeCurrent();
      g_markovQueueCount++;
   }

   g_markovCounts[fromState][toState]++;
   g_markovTradesRecorded++;
   RecomputeMarkovTransitionsFromCounts();
   g_lastMarkovState = toState;
}
//+------------------------------------------------------------------+
void LogMarkovAdjustmentSimulation()
{
   if(!INPUT_ENABLE_LOGGING)
      return;

   double simLossBoost = MathMin(MathMax(3.0 * (5 - 2), -15.0), 15.0);
   double simWinPenalty = MathMin(MathMax(-5.0 * (6 - 2), -15.0), 15.0);
   Print("MARKOV SIM: Example loss-side boost at 5 losses = ", simLossBoost,
         " | Example win-side penalty at 6 wins = ", simWinPenalty,
         " | Target clamp range [-15, +15]");
}
//+------------------------------------------------------------------+
double GetMarkovConfidenceAdjustment()
{
   const int minimumObservations = 10;
   const int minimumRowSamples = 3;

   if(!INPUT_ENABLE_MARKOV || g_markovQueueCount < minimumObservations)
      return 0;

   int totalSamples = 0;
   for(int r = 0; r < MARKOV_STATES; r++)
      totalSamples += GetMarkovRowTotal((ENUM_MARKOV_STATE)r);
   double effectiveSampleSize = (double)totalSamples / (double)MathMax(1, MARKOV_STATES * MARKOV_STATES);
   if(effectiveSampleSize < (double)minimumRowSamples)
      return 0;

   double adjustment = 0;

   if(g_consecutiveWins >= 3 && HasMarkovRowEvidence(MARKOV_WIN, minimumRowSamples))
   {
      double pLoss = g_markovTransitions[MARKOV_WIN][MARKOV_LOSS];
      int rowTotal = GetMarkovRowTotal(MARKOV_WIN);
      double certainty = MathMin(1.0, (double)rowTotal / (double)MathMax(minimumRowSamples * 4, 1));
      double shrunk = pLoss * certainty;
      if(shrunk > 0.4)
         adjustment = -5.0 * (g_consecutiveWins - 2) * certainty;
   }

   if(g_consecutiveLosses >= 3 && HasMarkovRowEvidence(MARKOV_LOSS, minimumRowSamples))
   {
      double pWin = g_markovTransitions[MARKOV_LOSS][MARKOV_WIN];
      int rowTotal = GetMarkovRowTotal(MARKOV_LOSS);
      double certainty = MathMin(1.0, (double)rowTotal / (double)MathMax(minimumRowSamples * 4, 1));
      double shrunk = pWin * certainty;
      if(shrunk > 0.3)
         adjustment += 3.0 * (g_consecutiveLosses - 2) * certainty;
   }

   adjustment = MathMax(-15.0, MathMin(15.0, adjustment));

   static bool loggedSimulation = false;
   if(!loggedSimulation)
   {
      LogMarkovAdjustmentSimulation();
      loggedSimulation = true;
   }

   return adjustment;
}
//+------------------------------------------------------------------+
//+------------------------------------------------------------------+
//| SECTION 17: ML & FINGERPRINT SYSTEM (Part 3)                     |
//+------------------------------------------------------------------+
double GetMLConfidenceMultiplier(const string &combination)
{
   for(int i = 0; i < g_combinationStatsCount; i++)
   {
      if(g_combinationStats[i].combination == combination)
      {
         int minTrades = INPUT_ENABLE_COMBINATION_ADAPTIVE ? INPUT_COMBO_MIN_TRADES : INPUT_MIN_TRADES_FOR_ML;
         if(g_combinationStats[i].totalTrades >= minTrades)
            return g_combinationStats[i].confidenceMultiplier;
         else
            return 1.0; // Not enough data
      }
   }
   return 1.0; // Unknown combination
}
//+------------------------------------------------------------------+
double GetFingerprintTimeDecay(const datetime lastSeen)
{
   if(lastSeen <= 0) return 1.0;
   double elapsedDays = (double)MathMax(0, TimeCurrent() - lastSeen) / 86400.0;
   double lambda = MathMax(0.001, 1.0 - INPUT_LEARNING_DECAY);
   return MathExp(-lambda * elapsedDays);
}

double GetFingerprintBoost(const string &fpId, const string &combination)
{
   for(int i = 0; i < g_fingerprintCount; i++)
   {
      if(g_fingerprints[i].id == fpId)
      {
         if(g_fingerprints[i].totalOccurrences >= 5)
         {
            double winRate = g_fingerprints[i].winRate;
            double decay = GetFingerprintTimeDecay(g_fingerprints[i].lastSeen);
            if(winRate >= 0.7) return 10.0 * decay;
            else if(winRate >= 0.6) return 5.0 * decay;
            else if(winRate >= 0.5) return 0.0;
            else if(winRate >= 0.4) return -5.0 * decay;
            else return -10.0 * decay;
         }
      }
   }
   return 0; // Unknown fingerprint
}
//+------------------------------------------------------------------+
string GenerateFingerprint(const SignalResult &signals, int session, int dayOfWeek)
{
   return signals.combinationString + "_S" + IntegerToString(session) +
          "_D" + IntegerToString(dayOfWeek) + "_R" + IntegerToString((int)g_currentRegime);
}
//+------------------------------------------------------------------+


string BuildComboFromMask(int mask, const string &names[], int n)
{
   string combo = "";
   for(int i = 0; i < n; i++)
   {
      if((mask & (1 << i)) == 0) continue;
      if(StringLen(combo) > 0) combo += "_";
      combo += names[i];
   }
   return combo;
}

int IndicatorNameIndex(const string token)
{
   string factorNames[8] = {"EMA","RSI","STOCH","ENGULF","BREAK","VOL","MACD","WPR"};
   for(int i = 0; i < 8; i++)
      if(token == factorNames[i]) return i;
   return -1;
}

int ParseRawSignalCombinationOrdered(const string rawCombination, string &orderedIndicators[])
{
   ArrayResize(orderedIndicators, 0);
   if(StringLen(rawCombination) == 0) return 0;

   string factorNames[8] = {"EMA","RSI","STOCH","ENGULF","BREAK","VOL","MACD","WPR"};
   bool present[8];
   ArrayInitialize(present, false);

   string parts[];
   int cnt = StringSplit(rawCombination, '_', parts);
   for(int i = 0; i < cnt; i++)
   {
      int idx = IndicatorNameIndex(parts[i]);
      if(idx >= 0) present[idx] = true;
   }

   for(int i = 0; i < 8; i++)
   {
      if(!present[i]) continue;
      int n = ArraySize(orderedIndicators);
      ArrayResize(orderedIndicators, n + 1);
      orderedIndicators[n] = factorNames[i];
   }
   return ArraySize(orderedIndicators);
}

int BuildCanonicalComboSubsets(const string rawCombination, int k, string &subsets[])
{
   ArrayResize(subsets, 0);
   string orderedIndicators[];
   int active = ParseRawSignalCombinationOrdered(rawCombination, orderedIndicators);
   if(active < k || k <= 0) return 0;

   int maxMask = (1 << active);
   for(int mask = 1; mask < maxMask; mask++)
   {
      int bits = 0;
      for(int b = 0; b < active; b++) if((mask & (1 << b)) != 0) bits++;
      if(bits != k) continue;

      string combo = BuildComboFromMask(mask, orderedIndicators, active);
      int n = ArraySize(subsets);
      ArrayResize(subsets, n + 1);
      subsets[n] = combo;
   }
   return ArraySize(subsets);
}

void BuildDeterministicComboUniverse()
{
   ArrayResize(g_comboUniverse, 0);
   g_comboUniverseCount = 0;
   g_comboObservedCount = 0;

   if(!INPUT_ENABLE_FULL_COMBO_UNIVERSE)
      return;

   int totalSignals = MathMin(INPUT_TOTAL_SIGNALS, INPUT_TOTAL_SIGNAL_FACTORS);
   int k = INPUT_MIN_SIGNALS;
   string factorNames[8] = {"EMA","RSI","STOCH","ENGULF","BREAK","VOL","MACD","WPR"};

   for(int mask = 1; mask < (1 << totalSignals); mask++)
   {
      int bits = 0;
      for(int b = 0; b < totalSignals; b++) if((mask & (1 << b)) != 0) bits++;
      if(bits != k) continue;
      string combo = BuildComboFromMask(mask, factorNames, totalSignals);
      int idx = ArraySize(g_comboUniverse);
      ArrayResize(g_comboUniverse, idx + 1);
      g_comboUniverse[idx] = combo;
   }

   g_comboUniverseCount = ArraySize(g_comboUniverse);
   if(INPUT_ENABLE_LOGGING)
      Print("COMBO UNIVERSE INIT: totalSignals=", totalSignals, " k=", k, " totalCombos=", g_comboUniverseCount);
}

int FindCombinationIndex(const string combo)
{
   for(int i = 0; i < g_combinationStatsCount; i++)
      if(g_combinationStats[i].combination == combo) return i;
   return -1;
}

double ComputeEntropyFromCounts(int wins, int losses)
{
   int total = wins + losses;
   if(total <= 0 || wins <= 0 || losses <= 0) return 0.0;
   double pW = (double)wins / total;
   double pL = (double)losses / total;
   return -(pW * MathLog(pW) / MathLog(2.0) + pL * MathLog(pL) / MathLog(2.0));
}

void SaveCombinationStatsSnapshot()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_combo_stats.csv";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;

   FileWrite(handle, "ComboID", "Combination", "Seen", "ObservedTrades", "Wins", "Losses", "PF", "Expectancy", "Strength", "Entropy", "InfoGain", "RankScore");
   for(int i = 0; i < g_combinationStatsCount; i++)
   {
      FileWrite(handle,
                g_combinationStats[i].comboId,
                g_combinationStats[i].combination,
                g_combinationStats[i].seen ? 1 : 0,
                g_combinationStats[i].totalTrades,
                g_combinationStats[i].wins,
                g_combinationStats[i].losses,
                g_combinationStats[i].profitFactor,
                g_combinationStats[i].expectancy,
                g_combinationStats[i].strengthScore,
                g_combinationStats[i].entropy,
                g_combinationStats[i].infoGain,
                g_combinationStats[i].rankScore);
   }
   FileClose(handle);
   if(!ReplaceFileAtomic(tmpName, filename))
      Print("WARNING: Failed atomic replace for combination stats file ", filename);
}

void RecomputeCombinationDerivedMetrics(int idx)
{
   if(idx < 0 || idx >= g_combinationStatsCount) return;
   if(g_combinationStats[idx].totalTrades <= 0) return;

   g_combinationStats[idx].winRate = (double)g_combinationStats[idx].wins / g_combinationStats[idx].totalTrades;
   g_combinationStats[idx].profitFactor = (g_combinationStats[idx].totalLoss > 0.0) ?
      g_combinationStats[idx].totalProfit / g_combinationStats[idx].totalLoss :
      (g_combinationStats[idx].totalProfit > 0.0 ? 10.0 : 0.0);
   g_combinationStats[idx].avgProfit = (g_combinationStats[idx].wins > 0) ?
      g_combinationStats[idx].totalProfit / g_combinationStats[idx].wins : 0.0;
   g_combinationStats[idx].avgLoss = (g_combinationStats[idx].losses > 0) ?
      g_combinationStats[idx].totalLoss / g_combinationStats[idx].losses : 0.0;
   g_combinationStats[idx].expectancy = (g_combinationStats[idx].totalTrades > 0) ?
      ((g_combinationStats[idx].totalProfit - g_combinationStats[idx].totalLoss) / g_combinationStats[idx].totalTrades) : 0.0;

   double score = 50.0;
   double wrComponent = MathMax(MathMin((g_combinationStats[idx].winRate - 0.5) * 100.0, 30.0), -30.0);
   double pfComponent = MathMax(MathMin((g_combinationStats[idx].profitFactor - 1.0) * 20.0, 25.0), -25.0);
   score = MathMax(MathMin(score + wrComponent + pfComponent, 100.0), 0.0);
   g_combinationStats[idx].strengthScore = score;

   double baselineEntropy = 1.0;
   g_combinationStats[idx].entropy = ComputeEntropyFromCounts(g_combinationStats[idx].wins, g_combinationStats[idx].losses);
   g_combinationStats[idx].infoGain = MathMax(0.0, baselineEntropy - g_combinationStats[idx].entropy);
   double igNormalized = MathMax(0.0, MathMin(g_combinationStats[idx].infoGain / baselineEntropy, 1.0));
   if(g_combinationStats[idx].totalTrades < INPUT_COMBO_MIN_TRADES)
      igNormalized *= ((double)g_combinationStats[idx].totalTrades / MathMax(1, INPUT_COMBO_MIN_TRADES));

   if(INPUT_COMBO_RANK_MODE == COMBO_RANK_ENTROPY_IG)
      g_combinationStats[idx].rankScore = igNormalized * 100.0;
   else if(INPUT_COMBO_RANK_MODE == COMBO_RANK_HYBRID)
      g_combinationStats[idx].rankScore = (score * 0.6) + (igNormalized * 100.0 * 0.4);
   else
      g_combinationStats[idx].rankScore = score;

   g_combinationStats[idx].confidenceMultiplier = MathMax(MathMin(1.0 + (g_combinationStats[idx].rankScore - 50.0) / 200.0, 1.5), 0.5);
}

void UpdateCombinationStatsIncremental(const TrainingData &row)
{
   string canonicalSubsets[];
   int subsetCount = BuildCanonicalComboSubsets(row.signalCombination, INPUT_MIN_SIGNALS, canonicalSubsets);
   if(subsetCount <= 0) return;

   for(int si = 0; si < subsetCount; si++)
   {
      int idx = -1;
      for(int i = 0; i < g_combinationStatsCount; i++)
         if(g_combinationStats[i].combination == canonicalSubsets[si]) { idx = i; break; }

      if(idx < 0 && g_combinationStatsCount < MAX_COMBINATION_STATS)
      {
         idx = g_combinationStatsCount++;
         ZeroMemory(g_combinationStats[idx]);
         g_combinationStats[idx].combination = canonicalSubsets[si];
         g_combinationStats[idx].comboId = canonicalSubsets[si];
      }
      if(idx < 0) continue;

      g_combinationStats[idx].seen = true;
      g_combinationStats[idx].totalTrades++;
      if(row.isWin) { g_combinationStats[idx].wins++; g_combinationStats[idx].totalProfit += row.profitLoss; }
      else { g_combinationStats[idx].losses++; g_combinationStats[idx].totalLoss += MathAbs(row.profitLoss); }

      if(row.entrySession == 0) { g_combinationStats[idx].asianTotal++; if(row.isWin) g_combinationStats[idx].asianWins++; }
      else if(row.entrySession == 1) { g_combinationStats[idx].londonTotal++; if(row.isWin) g_combinationStats[idx].londonWins++; }
      else if(row.entrySession == 2) { g_combinationStats[idx].nyTotal++; if(row.isWin) g_combinationStats[idx].nyWins++; }

      if(row.entryRegime == REGIME_TRENDING) { g_combinationStats[idx].trendingTotal++; if(row.isWin) g_combinationStats[idx].trendingWins++; }
      else if(row.entryRegime == REGIME_RANGING) { g_combinationStats[idx].rangingTotal++; if(row.isWin) g_combinationStats[idx].rangingWins++; }

      RecomputeCombinationDerivedMetrics(idx);
   }
}
void RecalculateCombinationStats()
{
   // Reset stats
   g_combinationStatsCount = 0;

   if(INPUT_ENABLE_FULL_COMBO_UNIVERSE)
   {
      for(int u = 0; u < g_comboUniverseCount && g_combinationStatsCount < MAX_COMBINATION_STATS; u++)
      {
         int idx = g_combinationStatsCount++;
         ZeroMemory(g_combinationStats[idx]);
         g_combinationStats[idx].combination = g_comboUniverse[u];
         g_combinationStats[idx].comboId = g_comboUniverse[u];
         g_combinationStats[idx].seen = false;
      }
   }

   // Group trades by canonical size-k subsets extracted from raw combinations
   for(int i = 0; i < g_trainingDataCount; i++)
   {
      string canonicalSubsets[];
      int subsetCount = BuildCanonicalComboSubsets(g_trainingData[i].signalCombination, INPUT_MIN_SIGNALS, canonicalSubsets);
      for(int si = 0; si < subsetCount; si++)
      {
         int idx = -1;
         for(int j = 0; j < g_combinationStatsCount; j++)
            if(g_combinationStats[j].combination == canonicalSubsets[si]) { idx = j; break; }

         if(idx < 0 && g_combinationStatsCount < MAX_COMBINATION_STATS)
         {
            idx = g_combinationStatsCount;
            g_combinationStatsCount++;
            ZeroMemory(g_combinationStats[idx]);
            g_combinationStats[idx].combination = canonicalSubsets[si];
            g_combinationStats[idx].comboId = canonicalSubsets[si];
         }

         if(idx < 0) continue;

         g_combinationStats[idx].seen = true;
         g_combinationStats[idx].totalTrades++;
         if(g_trainingData[i].isWin)
         {
            g_combinationStats[idx].wins++;
            g_combinationStats[idx].totalProfit += g_trainingData[i].profitLoss;
         }
         else
         {
            g_combinationStats[idx].losses++;
            g_combinationStats[idx].totalLoss += MathAbs(g_trainingData[i].profitLoss);
         }

         if(g_trainingData[i].entrySession == 0)
         {
            g_combinationStats[idx].asianTotal++;
            if(g_trainingData[i].isWin) g_combinationStats[idx].asianWins++;
         }
         else if(g_trainingData[i].entrySession == 1)
         {
            g_combinationStats[idx].londonTotal++;
            if(g_trainingData[i].isWin) g_combinationStats[idx].londonWins++;
         }
         else if(g_trainingData[i].entrySession == 2)
         {
            g_combinationStats[idx].nyTotal++;
            if(g_trainingData[i].isWin) g_combinationStats[idx].nyWins++;
         }

         if(g_trainingData[i].entryRegime == REGIME_TRENDING)
         {
            g_combinationStats[idx].trendingTotal++;
            if(g_trainingData[i].isWin) g_combinationStats[idx].trendingWins++;
         }
         else if(g_trainingData[i].entryRegime == REGIME_RANGING)
         {
            g_combinationStats[idx].rangingTotal++;
            if(g_trainingData[i].isWin) g_combinationStats[idx].rangingWins++;
         }
      }
   }

   // Derive metrics
   g_comboObservedCount = 0;
   for(int i = 0; i < g_combinationStatsCount; i++)
   {
      if(g_combinationStats[i].totalTrades > 0) g_comboObservedCount++;
      RecomputeCombinationDerivedMetrics(i);
   }

   if(INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_LOG_COMBINATION_INSIGHTS)
   {
      int topN = MathMax(1, INPUT_COMBO_INSIGHT_TOP_N);
      int bestIdx = -1;
      int worstIdx = -1;
      for(int i = 0; i < g_combinationStatsCount; i++)
      {
         if(g_combinationStats[i].totalTrades < INPUT_COMBO_MIN_TRADES) continue;
         if(bestIdx < 0 || g_combinationStats[i].rankScore > g_combinationStats[bestIdx].rankScore)
            bestIdx = i;
         if(worstIdx < 0 || g_combinationStats[i].rankScore < g_combinationStats[worstIdx].rankScore)
            worstIdx = i;
      }

      if(bestIdx >= 0)
      {
         Print("COMBO STRENGTH: Best=", g_combinationStats[bestIdx].combination,
               " | Trades=", g_combinationStats[bestIdx].totalTrades,
               " | Rank=", DoubleToString(g_combinationStats[bestIdx].rankScore, 1),
               " | WR=", DoubleToString(g_combinationStats[bestIdx].winRate * 100.0, 1), "%",
               " | PF=", DoubleToString(g_combinationStats[bestIdx].profitFactor, 2));
      }

      if(worstIdx >= 0)
      {
         Print("COMBO WEAKNESS: Worst=", g_combinationStats[worstIdx].combination,
               " | Trades=", g_combinationStats[worstIdx].totalTrades,
               " | Rank=", DoubleToString(g_combinationStats[worstIdx].rankScore, 1),
               " | WR=", DoubleToString(g_combinationStats[worstIdx].winRate * 100.0, 1), "%",
               " | PF=", DoubleToString(g_combinationStats[worstIdx].profitFactor, 2));
      }
   }

   Print("COMBO COVERAGE: observed=", g_comboObservedCount, " / total=", MathMax(g_combinationStatsCount, 1));
   RebuildDecisionTreeFeatureModule();
   SaveCombinationStatsSnapshot();
}

//+------------------------------------------------------------------+
//| SECTION 17B: DECISION-TREE FEATURE MODULE                        |
//+------------------------------------------------------------------+
void SaveTreeFeatureRanking()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_tree_feature_rank.csv";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;

   FileWrite(handle, "feature", "support", "yesWins", "yesLosses", "noWins", "noLosses", "entropyYes", "entropyNo", "IG", "selectedFlag");
   for(int i = 0; i < g_treeFeatureMetricCount; i++)
   {
      int yesTotal = g_treeFeatureMetrics[i].yesWins + g_treeFeatureMetrics[i].yesLosses;
      FileWrite(handle,
                g_treeFeatureMetrics[i].feature,
                yesTotal,
                g_treeFeatureMetrics[i].yesWins,
                g_treeFeatureMetrics[i].yesLosses,
                g_treeFeatureMetrics[i].noWins,
                g_treeFeatureMetrics[i].noLosses,
                g_treeFeatureMetrics[i].entropyYes,
                g_treeFeatureMetrics[i].entropyNo,
                g_treeFeatureMetrics[i].infoGain,
                g_treeFeatureMetrics[i].selected ? 1 : 0);
   }
   FileClose(handle);
   if(!ReplaceFileAtomic(tmpName, filename))
      Print("WARNING: Failed atomic replace for tree feature metrics file ", filename);
}

bool ComboSubsetContainsFeature(const string feature, const string &canonicalSubsets[])
{
   for(int i = 0; i < ArraySize(canonicalSubsets); i++)
      if(canonicalSubsets[i] == feature) return true;
   return false;
}

int CountSelectedTreeFeatureMatches(const string rawCombination)
{
   if(g_treeSelectedFeatureCount <= 0) return 0;
   string canonicalSubsets[];
   int subsetCount = BuildCanonicalComboSubsets(rawCombination, INPUT_MIN_SIGNALS, canonicalSubsets);
   if(subsetCount <= 0) return 0;

   int matches = 0;
   for(int i = 0; i < g_treeSelectedFeatureCount; i++)
      if(ComboSubsetContainsFeature(g_treeSelectedFeatures[i], canonicalSubsets)) matches++;
   return matches;
}

double GetTreeConfidenceAdjustment(const string rawCombination)
{
   if(!INPUT_ENABLE_TREE_FEATURE_MODULE || !INPUT_TREE_ADJUST_CONFIDENCE_ON || g_treeSelectedFeatureCount <= 0)
      return 0.0;

   string canonicalSubsets[];
   int subsetCount = BuildCanonicalComboSubsets(rawCombination, INPUT_MIN_SIGNALS, canonicalSubsets);
   if(subsetCount <= 0) return 0.0;

   double cumulative = 0.0;
   int hits = 0;
   for(int i = 0; i < g_treeFeatureMetricCount; i++)
   {
      if(!g_treeFeatureMetrics[i].selected) continue;
      if(!ComboSubsetContainsFeature(g_treeFeatureMetrics[i].feature, canonicalSubsets)) continue;

      int yesTot = g_treeFeatureMetrics[i].yesWins + g_treeFeatureMetrics[i].yesLosses;
      if(yesTot <= 0) continue;
      double wrEdge = ((double)g_treeFeatureMetrics[i].yesWins / yesTot) - 0.5;
      cumulative += wrEdge * g_treeFeatureMetrics[i].infoGain;
      hits++;
   }

   if(hits <= 0) return 0.0;
   return cumulative * 100.0 * INPUT_TREE_CONFIDENCE_WEIGHT;
}

void RebuildDecisionTreeFeatureModule()
{
   g_treeParentEntropy = 0.0;
   g_treeFeatureMetricCount = 0;
   g_treeSelectedFeatureCount = 0;
   ArrayResize(g_treeFeatureMetrics, 0);
   ArrayResize(g_treeSelectedFeatures, 0);

   if(!INPUT_ENABLE_TREE_FEATURE_MODULE || g_trainingDataCount <= 0) return;

   int parentWins = 0;
   int parentLosses = 0;
   for(int i = 0; i < g_trainingDataCount; i++)
   {
      if(g_trainingData[i].isWin) parentWins++;
      else parentLosses++;
   }
   g_treeParentEntropy = ComputeEntropyFromCounts(parentWins, parentLosses);

   int branchMin = MathMax(INPUT_COMBO_MIN_TRADES, INPUT_TREE_BRANCH_MIN_SUPPORT);
   int featureUniverse = INPUT_ENABLE_FULL_COMBO_UNIVERSE ? g_comboUniverseCount : g_combinationStatsCount;
   for(int f = 0; f < featureUniverse; f++)
   {
      string feature = INPUT_ENABLE_FULL_COMBO_UNIVERSE ? g_comboUniverse[f] : g_combinationStats[f].combination;
      if(StringLen(feature) == 0) continue;

      int yesWins = 0, yesLosses = 0, noWins = 0, noLosses = 0;
      for(int i = 0; i < g_trainingDataCount; i++)
      {
         string canonicalSubsets[];
         int subsetCount = BuildCanonicalComboSubsets(g_trainingData[i].signalCombination, INPUT_MIN_SIGNALS, canonicalSubsets);
         bool hasFeature = (subsetCount > 0 && ComboSubsetContainsFeature(feature, canonicalSubsets));
         if(hasFeature)
         {
            if(g_trainingData[i].isWin) yesWins++;
            else yesLosses++;
         }
         else
         {
            if(g_trainingData[i].isWin) noWins++;
            else noLosses++;
         }
      }

      int yesTotal = yesWins + yesLosses;
      int noTotal = noWins + noLosses;
      if(yesTotal < branchMin || noTotal < branchMin) continue;

      double hYes = ComputeEntropyFromCounts(yesWins, yesLosses);
      double hNo = ComputeEntropyFromCounts(noWins, noLosses);
      double total = yesTotal + noTotal;
      double weighted = ((double)yesTotal / total) * hYes + ((double)noTotal / total) * hNo;
      double ig = MathMax(0.0, g_treeParentEntropy - weighted);

      int idx = ArraySize(g_treeFeatureMetrics);
      ArrayResize(g_treeFeatureMetrics, idx + 1);
      g_treeFeatureMetrics[idx].feature = feature;
      g_treeFeatureMetrics[idx].support = yesTotal;
      g_treeFeatureMetrics[idx].yesWins = yesWins;
      g_treeFeatureMetrics[idx].yesLosses = yesLosses;
      g_treeFeatureMetrics[idx].noWins = noWins;
      g_treeFeatureMetrics[idx].noLosses = noLosses;
      g_treeFeatureMetrics[idx].entropyYes = hYes;
      g_treeFeatureMetrics[idx].entropyNo = hNo;
      g_treeFeatureMetrics[idx].infoGain = ig;
      g_treeFeatureMetrics[idx].selected = false;
   }

   g_treeFeatureMetricCount = ArraySize(g_treeFeatureMetrics);
   for(int pick = 0; pick < INPUT_TREE_MAX_SELECTED_FEATURES; pick++)
   {
      int best = -1;
      for(int i = 0; i < g_treeFeatureMetricCount; i++)
      {
         if(g_treeFeatureMetrics[i].selected) continue;
         if(g_treeFeatureMetrics[i].infoGain < INPUT_TREE_MIN_IG) continue;
         if(best < 0 || g_treeFeatureMetrics[i].infoGain > g_treeFeatureMetrics[best].infoGain)
            best = i;
      }
      if(best < 0) break;

      g_treeFeatureMetrics[best].selected = true;
      int n = ArraySize(g_treeSelectedFeatures);
      ArrayResize(g_treeSelectedFeatures, n + 1);
      g_treeSelectedFeatures[n] = g_treeFeatureMetrics[best].feature;
      g_treeSelectedFeatureCount++;
   }

   SaveTreeFeatureRanking();
}

//+------------------------------------------------------------------+
//| SECTION 18: MARKET REGIME DETECTION                              |
//+------------------------------------------------------------------+
void DetectMarketRegime()
{
   double adx[];
   if(CopyBuffer(g_hADX_H1, 0, 0, 1, adx) < 1)
   {
      g_currentRegime = REGIME_UNKNOWN;
      return;
   }

   double volRatio = CalculateVolatilityRatio();

   // Volatility first
   if(volRatio >= 1.5)
   {
      g_currentRegime = REGIME_VOLATILE;
      return;
   }
   if(volRatio < 0.7)
   {
      g_currentRegime = REGIME_QUIET;
      return;
   }

      if(adx[0] >= 25)
      g_currentRegime = REGIME_TRENDING;
   else if(adx[0] < 20)
      g_currentRegime = REGIME_RANGING;
   else
      g_currentRegime = REGIME_UNKNOWN;
}
//+------------------------------------------------------------------+
//| SECTION 19: DECISION PIPELINE - FIXED!                           |
//+------------------------------------------------------------------+
bool RunDecisionPipeline(DecisionResult &decision)
{
g_mtfReadFailureThisTick = false;

   //--- STEP 1: Pre?trade gates
   string rejectReason = "";
   if(!CheckAllGates(rejectReason))
   {
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Gate rejected: " + rejectReason);
      return false;
   }

   //--- STEP 2: Signal detection
   SignalResult signals;
   ZeroMemory(signals);
   bool signalsDetected = DetectSignals(signals);
   if(INPUT_GATE_SIGNAL_DETECTION_ON && !signalsDetected)
   {
      g_gateDiagnostics.signalsRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Signal detection failed");
      return false;
   }
   if(!signalsDetected)
   {
      signals.totalSignals = INPUT_MIN_SIGNALS;
      signals.bullVotes = 1;
      signals.bearVotes = 0;
      signals.combinationString = "SIGNAL_GATE_BYPASS";
   }

   if(INPUT_GATE_MIN_SIGNALS_ON && signals.totalSignals < INPUT_MIN_SIGNALS)
   {
      g_gateDiagnostics.signalsRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Not enough signals: " + IntegerToString(signals.totalSignals) + " < " + IntegerToString(INPUT_MIN_SIGNALS));
      return false;
   }

   //--- STEP 3: Determine direction (base votes + MTF consensus weighting)
   int weightedBullVotes = signals.bullVotes;
   int weightedBearVotes = signals.bearVotes;

   int mtfConsensusStrength = 0;
   int mtfConsensusDirection = GetTimeframeDirectionConsensus(mtfConsensusStrength);
   if(INPUT_GATE_MTF_WEIGHTING_ON && mtfConsensusDirection == 1)
      weightedBullVotes += (int)MathRound(INPUT_MTF_CONSENSUS_VOTE_WEIGHT);
   else if(INPUT_GATE_MTF_WEIGHTING_ON && mtfConsensusDirection == -1)
      weightedBearVotes += (int)MathRound(INPUT_MTF_CONSENSUS_VOTE_WEIGHT);

   int direction = 0;
   if(weightedBullVotes > weightedBearVotes)
   {
      direction = 1;
   }
   else if(weightedBearVotes > weightedBullVotes)
   {
      direction = -1;
   }
   else
   {
      g_gateDiagnostics.signalsRejects++;
      if(INPUT_ENABLE_LOGGING)
         Print("Signal tie after MTF weighting: baseBull=", signals.bullVotes,
               " baseBear=", signals.bearVotes,
               " weightedBull=", weightedBullVotes,
               " weightedBear=", weightedBearVotes,
               " consensusDir=", mtfConsensusDirection,
               " consensusFrames=", mtfConsensusStrength);
      return false;
   }

   if(INPUT_ENABLE_LOGGING)
      Print("Votes: baseBull=", signals.bullVotes,
            " baseBear=", signals.bearVotes,
            " weightedBull=", weightedBullVotes,
            " weightedBear=", weightedBearVotes,
            " consensusDir=", mtfConsensusDirection,
            " consensusFrames=", mtfConsensusStrength);

   //--- STEP 4: ADX filter (optional)
   if(INPUT_GATE_ADX_FILTER_ON && INPUT_USE_ADX_FILTER)
   {
      double adx[];
      int adxRead = CopyBuffer(g_hADX_M1, 0, 0, 1, adx);
      if(adxRead != 1)
      {
         g_gateDiagnostics.adxDataReadRejects++;
         if(INPUT_ENABLE_LOGGING)
            LogWithRestartGuard("ADX data unavailable in RunDecisionPipeline (CopyBuffer=" + IntegerToString(adxRead) + ") - rejecting entry");
         return false;
      }

      if(adx[0] < INPUT_ADX_MIN_THRESHOLD)
      {
         if(INPUT_ENABLE_LOGGING)
            LogWithRestartGuard("ADX filter failed: " + DoubleToString(adx[0], 2) + " < " + DoubleToString(INPUT_ADX_MIN_THRESHOLD, 2));
         return false;
      }
    }

    //--- STEP 4b: Same-direction limit check
    string dirReject = "";
    if(INPUT_GATE_SAME_DIRECTION_ON && !CheckSameDirectionLimit(direction, dirReject))
    {
       if(INPUT_ENABLE_LOGGING)
          LogWithRestartGuard("Same-direction limit: " + dirReject);
       return false;
    }
    int sameDirectionRemainingSec = 0;
    if(INPUT_GATE_SAME_DIRECTION_ON && IsSameDirectionCooldownActive(direction, sameDirectionRemainingSec))
    {
       g_gateDiagnostics.cooldownRejects++;
       string dirText = (direction == 1 ? "BUY" : "SELL");
       if(INPUT_ENABLE_LOGGING)
          LogWithRestartGuard("Same-direction cooldown active (" + dirText + ", " + IntegerToString(sameDirectionRemainingSec) + "s remaining)");
       return false;
    }

    //--- STEP 4c: Proximity check
    string proxReject = "";
    if(INPUT_GATE_PROXIMITY_ON && !CheckProximity(direction, proxReject))
    {
       if(INPUT_ENABLE_LOGGING)
          LogWithRestartGuard("Proximity reject: " + proxReject);
       return false;
    }

    //--- STEP 5: MTF alignment (optional)
    int mtfScore = CalculateMTFAlignment(direction);
    if(INPUT_GATE_MTF_ALIGNMENT_ON && INPUT_MIN_MTF_SCORE > 0 && mtfScore < INPUT_MIN_MTF_SCORE)
    {
       g_gateDiagnostics.mtfRejects++;
       if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("MTF alignment failed: " + IntegerToString(mtfScore) + " < " + IntegerToString(INPUT_MIN_MTF_SCORE) +
                             " | dataReadFailedThisTick=" + (g_mtfReadFailureThisTick ? "YES" : "NO"));
                                    return false;
    }

   //--- STEP 6: Fingerprint generation
   int sessionNow = GetCurrentSession();
   MqlDateTime dtNow;
   TimeToStruct(TimeCurrent(), dtNow);
   string fpId = GenerateFingerprint(signals, sessionNow, dtNow.day_of_week);

   //--- STEP 7: Threat calculation
   double threat = CalculateMarketThreat();
   ENUM_THREAT_ZONE threatZone = GetThreatZone(threat);

   // Soft gate near threshold, hard gate only when materially above threshold
   if(INPUT_GATE_THREAT_HARD_BLOCK_ON && INPUT_THREAT_HARD_ENTRY_BLOCK_ON && g_effThreatHardBlock && threat > (INPUT_MAX_THREAT_ENTRY + 10.0))
   {
      g_gateDiagnostics.threatRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Threat hard-block: " + DoubleToString(threat, 2) + " > " + DoubleToString(INPUT_MAX_THREAT_ENTRY + 10.0, 2));
      return false;
   }

   //--- STEP 8: Confidence calculation
   double confidence = CalculateConfidence(signals, direction, mtfScore,
                                            fpId, signals.combinationString, threat);

   if(!IsFiniteInRange(confidence, 0.0, 100.0) || !IsFiniteInRange(threat, 0.0, 100.0))
   {
      RegisterDataWarning("NaN/Inf model output in confidence/threat");
      return false;
   }

   // Apply adaptive minimum confidence
   double minConf = g_adaptive.minConfThreshold;
   if(INPUT_GATE_CONFIDENCE_MIN_ON && confidence < minConf)
   {
      g_gateDiagnostics.confidenceRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Confidence too low: " + DoubleToString(confidence, 2) + " < " + DoubleToString(minConf, 2));
      return false;
   }

   if(INPUT_ENABLE_TREE_FEATURE_MODULE && INPUT_TREE_ENTRY_GATE_ON)
   {
      int selectedHits = CountSelectedTreeFeatureMatches(signals.combinationString);
      if(selectedHits < INPUT_TREE_MIN_SELECTED_MATCH)
      {
         if(INPUT_ENABLE_LOGGING)
            LogWithRestartGuard("Tree feature gate failed: hits=" + IntegerToString(selectedHits) + " < " + IntegerToString(INPUT_TREE_MIN_SELECTED_MATCH));
         return false;
      }
   }

   //--- STEP 9: Q?Learning action selection (optional)
   int positions = CountMainPositionsFromBroker();
   double drawdown = CalculateDrawdownPercent();
   ENUM_RL_ACTION rlAction = ApplyRLToDecision(confidence, threat,
                                               positions, drawdown);

   // Honor RL skip decision
   if(INPUT_ENABLE_RL && INPUT_RL_INFERENCE_ON && rlAction == RL_SKIP_TRADE && g_rlTradesCompleted >= INPUT_RL_MIN_TRADES)
   {
      if(!g_rngSeeded)
      {
         int seed = (int)(TimeLocal() ^ (datetime)GetTickCount());
         MathSrand(seed);
         g_rngSeeded = true;
         Print("RNG SEEDED LATE: applied defensive seed in RunDecisionPipeline | seed=", seed);
      }

      // Randomised skip based on RL weight (default weight = 0.3)
      if((double)MathRand() / 32767.0 < INPUT_RL_WEIGHT)
      {
         if(INPUT_ENABLE_LOGGING)
            LogWithRestartGuard("RL decided to skip trade");
         return false;
      }
   }

   //--- STEP 10: Decision matrix - we now only block on extreme threat
   if(INPUT_GATE_THREAT_EXTREME_BLOCK_ON && g_effThreatExtremeZoneBlock && threatZone == THREAT_EXTREME)
   {
      g_gateDiagnostics.threatRejects++;
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("Extreme threat zone - blocking trade");
      return false;
   }

   //--- STEP 11: SL/TP calculation (threat?adjusted)
   double slPoints, tpPoints;
   if(!CalculateSLTP(direction, threat, slPoints, tpPoints))
   {
      if(INPUT_ENABLE_LOGGING)
         LogWithRestartGuard("SL/TP calculation failed");
      return false;
   }

   double expectedRR = (slPoints > 0.0) ? (tpPoints / slPoints) : 0.0;
   if(INPUT_GATE_EFFECTIVE_RR_ON && (!MathIsValidNumber(expectedRR) || expectedRR <= INPUT_MIN_EFFECTIVE_RR_AFTER_SPREAD))
   {
      RegisterDataWarning("Rejected by spread-adjusted RR gate");
      return false;
   }

   //--- STEP 12: Position sizing (threat + confidence + RL)
   double lotSize = CalculateLotSize(slPoints, confidence, threat, rlAction, direction);

   // Soft threat gating before hard no-trade: shrink lot near threat threshold
   if(INPUT_THREAT_SOFT_LOT_SHRINK_ON && g_effThreatSoftLotShrink && threat > INPUT_MAX_THREAT_ENTRY)
   {
      double over = threat - INPUT_MAX_THREAT_ENTRY;
      double threatSoftFactor = MathMax(0.25, 1.0 - (over / 10.0) * 0.5);
      lotSize *= threatSoftFactor;
   }
   if(INPUT_LOT_HIGH_ADX_BOOST_ON && INPUT_ENABLE_HIGH_ADX_RISK_MODE)
   {
      double adxNow[];
      if(CopyBuffer(g_hADX_M1, 0, 0, 1, adxNow) == 1 && adxNow[0] >= INPUT_HIGH_ADX_THRESHOLD)
         lotSize *= INPUT_HIGH_ADX_LOT_MULTIPLIER;
   }

if(INPUT_LOT_RISK_PARITY_CAP_ON && INPUT_ENABLE_RISK_PARITY_CAP)
   {
      int sess = GetCurrentSession();
      double volRatio = CalculateVolatilityRatio();
      double sessionWeight = (sess == 1) ? 1.0 : ((sess == 2) ? 0.9 : 0.7);
      double volNorm = MathMax(0.5, MathMin(2.0, volRatio));
      double capLots = INPUT_RISK_PARITY_BASE_CAP_LOTS * sessionWeight / volNorm;
      lotSize = MathMin(lotSize, MathMax(capLots, g_minLot));
   }

   // Final post-transform lot normalization (all multipliers/caps already applied)
   string volumeReason = "";
   double normalizedLotSize = 0.0;
   if(!NormalizeAndValidateOrderVolume(lotSize, normalizedLotSize, volumeReason))
   {
      if(INPUT_ENABLE_LOGGING)
      {
         Print("LOT NORMALIZATION REJECTED: requested=", DoubleToString(lotSize, g_lotDigits),
               " | reason=", volumeReason,
               " | step=", DoubleToString(GetEffectiveLotStep(), g_lotDigits),
               " | brokerMin=", DoubleToString(g_minLot, g_lotDigits),
               " | brokerMax=", DoubleToString(g_maxLot, g_lotDigits));
      }
      return false;
   }
   lotSize = normalizedLotSize;

   if(lotSize <= 0)
   {
      if(INPUT_ENABLE_LOGGING)
         Print("Lot size calculation returned 0");
      return false;
   }


   //--- Fill decision structure
   decision.shouldTrade = true;
   decision.direction = direction;
   decision.confidence = confidence;
   decision.threatLevel = threat;
   decision.threatZone = threatZone;
   decision.mtfScore = mtfScore;
   decision.signalCount = signals.totalSignals;
   decision.signalCombination = signals.combinationString;
   decision.slPoints = slPoints;
   decision.tpPoints = tpPoints;
   decision.lotSize = lotSize;
   decision.fingerprintId = fpId;
   decision.rlAction = rlAction;

   if(INPUT_ENABLE_LOGGING)
      Print("DECISION: ", (direction == 1 ? "BUY" : "SELL"),
            " | Conf: ", confidence, " | Threat: ", threat,
            " | Signals: ", signals.totalSignals, " (", signals.combinationString, ")");

   return true;
}
//+------------------------------------------------------------------+
bool IsOrderCooldownActive(int &remainingSec)
{
   remainingSec = 0;
   if(INPUT_ORDER_COOLDOWN_SECONDS <= 0)
      return false;

   int elapsedSec = (int)(TimeCurrent() - g_lastOrderTime);
   remainingSec = MathMax(0, INPUT_ORDER_COOLDOWN_SECONDS - elapsedSec);
   return (remainingSec > 0);
}

bool IsSameDirectionCooldownActive(int direction, int &remainingSec)
{
   remainingSec = 0;
   if(INPUT_SAME_DIRECTION_BLOCK_SECONDS <= 0)
      return false;

   datetime lastDirTime = (direction == 1) ? g_lastBuyOrderTime : g_lastSellOrderTime;
   if(lastDirTime <= 0)
      return false;

   int elapsedSec = (int)(TimeCurrent() - lastDirTime);
   remainingSec = MathMax(0, INPUT_SAME_DIRECTION_BLOCK_SECONDS - elapsedSec);
   return (remainingSec > 0);
}

bool CheckAllGates(string &rejectReason)
{
   // V7.2 FIX (BUG 3): Emergency zero-guard - if maxPositions somehow reaches 0, reset to input default
   if(g_adaptive.maxPositions <= 0)
   {
      g_adaptive.maxPositions = INPUT_MAX_CONCURRENT_TRADES;
      Print("WARNING: maxPositions was 0 or negative! Reset to ", INPUT_MAX_CONCURRENT_TRADES);
   }

   int windowSec = MathMax(60, INPUT_DATA_WARNING_WINDOW_MINUTES * 60);
   if(g_dataWarningWindowStart > 0 && (TimeCurrent() - g_dataWarningWindowStart) > windowSec)
   {
      g_dataWarningWindowStart = TimeCurrent();
      g_dataIntegrityWarnings = 0;
   }
   if(INPUT_GATE_DATA_ANOMALY_KILLSWITCH_ON && g_dataIntegrityWarnings >= INPUT_DATA_WARNING_KILL_SWITCH)
   {
      rejectReason = "Anomaly kill-switch active";
      return false;
   }

   bool isTesterMode = (MQLInfoInteger(MQL_TESTER) != 0);
   if(INPUT_GATE_TERMINAL_CONNECTED_ON && !isTesterMode && !TerminalInfoInteger(TERMINAL_CONNECTED))
   {
      rejectReason = "Terminal disconnected";
      return false;
   }

   if(INPUT_GATE_AUTOTRADING_ALLOWED_ON && !MQLInfoInteger(MQL_TRADE_ALLOWED))
   {
      rejectReason = "AutoTrading disabled";
      return false;
   }

   if(INPUT_GATE_SESSION_ON && !IsAllowedSession())
   {
      g_gateDiagnostics.sessionRejects++;
      rejectReason = "Outside trading session";
      return false;
   }

   // Cooldown is enforced in ExecuteOrder so confirmed flip replacements can bypass when configured.

   if(INPUT_GATE_MAX_DAILY_TRADES_ON && g_daily.tradesPlaced >= INPUT_MAX_DAILY_TRADES)
   {
      rejectReason = "Daily trade limit reached";
      return false;
   }

   double dayLoss = g_daily.lossToday;
   double maxDayLoss = g_daily.dayStartBalance * (INPUT_DAILY_LOSS_LIMIT_PERCENT / 100.0);
   if(INPUT_GATE_DAILY_LOSS_ON && dayLoss >= maxDayLoss)
   {
      rejectReason = "Daily loss limit reached";
      return false;
   }

   if(INPUT_GATE_CONSECUTIVE_LOSS_ON && g_consecutiveLosses >= INPUT_MAX_CONSECUTIVE_LOSSES)
   {
      rejectReason = "Consecutive loss limit reached";
      return false;
   }

   if(INPUT_GATE_SPREAD_ON && IsSpreadHigh())
   {
      rejectReason = "Spread too high";
      return false;
   }

    int inputCap = MathMax(1, INPUT_MAX_CONCURRENT_TRADES);
   int adaptiveCap = MathMax(1, g_adaptive.maxPositions);
   int allowedAdaptiveDelta = (INPUT_ALLOW_ADAPTIVE_MAX_POSITION_EXPANSION ? 2 : 0);
   int adaptiveModeCap = MathMin(adaptiveCap, inputCap + allowedAdaptiveDelta);
   int effectiveMaxMain = ((INPUT_MAX_MAIN_HARD_CAP_ON || INPUT_STRICT_OPPOSITE_FLIP_MODE) ? inputCap : adaptiveModeCap);

   int openMain = CountMainPositionsFromBroker();
   int pendingMain = ((INPUT_EXECUTION_MODE == PENDING_STOP && IsFeatureEnabled("pending_orders")) ? CountMainPendingStopsAllDirections() : 0);
   int effectiveExposure = GetEffectiveMainExposureCount();
   int currentTotalPositions = CountAllOurPositions();
   if(INPUT_GATE_MAX_POSITIONS_ON && effectiveExposure >= effectiveMaxMain)
   {
      g_gateDiagnostics.maxPositionsRejects++;
      rejectReason = "Max main exposure reached (openMain=" + IntegerToString(openMain) +
                     " pendingMain=" + IntegerToString(pendingMain) +
                     " effectiveExposure=" + IntegerToString(effectiveExposure) +
                     " effectiveMaxMain=" + IntegerToString(effectiveMaxMain) +
                     " inputCap=" + IntegerToString(inputCap) +
                     " adaptiveCap=" + IntegerToString(adaptiveCap) +
                     " total=" + IntegerToString(currentTotalPositions) + ")";
      return false;
   }

   if(INPUT_GATE_EA_PROTECTION_STATE_ON && g_effGateProtectionBlock && (g_eaState == STATE_EXTREME_RISK || g_eaState == STATE_DRAWDOWN_PROTECT))
   {
      rejectReason = "EA in protection mode";
      return false;
   }

   return true;
}
//+------------------------------------------------------------------+
bool IsCountableForEntryGating(const string comment)
{
  return IsMainEntryComment(comment);
}

 // Same-direction limit: prevent all positions being in one direction
 bool CheckSameDirectionLimit(int direction, string &rejectReason)
 {
    int sameCount = 0;
    int total = PositionsTotal();
    for(int i = 0; i < total; i++)
    {
       ulong ticket = PositionGetTicket(i);
       if(ticket == 0) continue;
       if(!IsOurPosition(ticket)) continue;

       string comment = PositionGetString(POSITION_COMMENT);
       if(!IsCountableForEntryGating(comment)) continue;

       int posDir = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
       if(posDir == direction)
          sameCount++;
    }

    if(INPUT_EXECUTION_MODE == PENDING_STOP && IsFeatureEnabled("pending_orders"))
       sameCount += CountPendingStopsByDirection(direction);

    if(sameCount >= INPUT_MAX_SAME_DIRECTION)
    {
       rejectReason = "Same-direction limit reached (" + IntegerToString(sameCount) +
                      " " + (direction == 1 ? "BUY" : "SELL") +
                      (INPUT_EXECUTION_MODE == PENDING_STOP ? " positions+pending" : " positions") + ")";
       if(INPUT_ENABLE_LOGGING)
          Print("CHECK SAME DIRECTION: REJECT | direction=", (direction == 1 ? "BUY" : "SELL"),
                " | sameCount=", sameCount, " | limit=", INPUT_MAX_SAME_DIRECTION);
       return false;
    }

    if(INPUT_ENABLE_LOGGING)
       Print("CHECK SAME DIRECTION: ACCEPT | direction=", (direction == 1 ? "BUY" : "SELL"),
             " | sameCount=", sameCount, " | limit=", INPUT_MAX_SAME_DIRECTION);
    return true;
 }
 //+------------------------------------------------------------------+
 // Proximity check: don't open new trade near existing entries
 bool CheckProximity(int direction, string &rejectReason)
 {
    if(INPUT_PROXIMITY_POINTS <= 0) return true;

    double currentPrice = (direction == 1) ?
                          SymbolInfoDouble(_Symbol, SYMBOL_ASK) :
                          SymbolInfoDouble(_Symbol, SYMBOL_BID);
    double minDist = INPUT_PROXIMITY_POINTS * g_point;

    int total = PositionsTotal();
    for(int i = 0; i < total; i++)
    {
       ulong ticket = PositionGetTicket(i);
       if(ticket == 0) continue;
       if(!IsOurPosition(ticket)) continue;

       string comment = PositionGetString(POSITION_COMMENT);
       if(!IsCountableForEntryGating(comment)) continue;

       double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
       double dist = MathAbs(currentPrice - entryPrice);

       if(dist < minDist)
       {
          rejectReason = "Too close to existing position (dist=" +
                         DoubleToString(dist / g_point, 0) + " pts < " +
                         DoubleToString(INPUT_PROXIMITY_POINTS, 0) + " pts)";
          return false;
       }
    }
    return true;
 }
 //+------------------------------------------------------------------+
 // Position age timeout: close stale positions



 //+------------------------------------------------------------------+
bool IsValidHourValue(int hour)
{
   return (hour >= 0 && hour <= 23);
}
//+------------------------------------------------------------------+
void WarnInvalidSessionHour(const string label, int value)
{
   Print("WARNING: Invalid session hour for ", label, " = ", value, " (expected 0..23)");
}
//+------------------------------------------------------------------+
bool IsHourInWindow(int hour, int start, int end)
{
   if(!IsValidHourValue(hour) || !IsValidHourValue(start) || !IsValidHourValue(end))
      return false;

   if(start < end)
      return (hour >= start && hour < end);

   if(start > end)
      return (hour >= start || hour < end);

   // Explicit behavior for start==end: full-day enabled session window.
   return true;
}
//+------------------------------------------------------------------+
void ValidateSessionHourInputs()
{
   if(!IsValidHourValue(INPUT_ASIAN_START)) WarnInvalidSessionHour("INPUT_ASIAN_START", INPUT_ASIAN_START);
   if(!IsValidHourValue(INPUT_ASIAN_END)) WarnInvalidSessionHour("INPUT_ASIAN_END", INPUT_ASIAN_END);
   if(!IsValidHourValue(INPUT_LONDON_START)) WarnInvalidSessionHour("INPUT_LONDON_START", INPUT_LONDON_START);
   if(!IsValidHourValue(INPUT_LONDON_END)) WarnInvalidSessionHour("INPUT_LONDON_END", INPUT_LONDON_END);
   if(!IsValidHourValue(INPUT_NY_START)) WarnInvalidSessionHour("INPUT_NY_START", INPUT_NY_START);
   if(!IsValidHourValue(INPUT_NY_END)) WarnInvalidSessionHour("INPUT_NY_END", INPUT_NY_END);
}
//+------------------------------------------------------------------+
bool ValidateSessionHourConfig(string &err)
{
   if(!IsValidHourValue(INPUT_ASIAN_START)) { err = "INPUT_ASIAN_START must be in 0..23"; return false; }
   if(!IsValidHourValue(INPUT_ASIAN_END)) { err = "INPUT_ASIAN_END must be in 0..23"; return false; }
   if(!IsValidHourValue(INPUT_LONDON_START)) { err = "INPUT_LONDON_START must be in 0..23"; return false; }
   if(!IsValidHourValue(INPUT_LONDON_END)) { err = "INPUT_LONDON_END must be in 0..23"; return false; }
   if(!IsValidHourValue(INPUT_NY_START)) { err = "INPUT_NY_START must be in 0..23"; return false; }
   if(!IsValidHourValue(INPUT_NY_END)) { err = "INPUT_NY_END must be in 0..23"; return false; }
   return true;
}
//+------------------------------------------------------------------+
void LogWithRestartGuard(const string message)
{
   Print(message);

   if(INPUT_REPEAT_LOG_RESTART_THRESHOLD <= 0)
      return;

   if(message == g_lastLogMessage)
      g_lastLogRepeatCount++;
   else
   {
      g_lastLogMessage = message;
      g_lastLogRepeatCount = 1;
   }

   if(g_lastLogRepeatCount >= INPUT_REPEAT_LOG_RESTART_THRESHOLD)
   {
      Print("RESTART GUARD: repeated log detected | message=", message,
            " | repeats=", g_lastLogRepeatCount,
            " | threshold=", INPUT_REPEAT_LOG_RESTART_THRESHOLD);

      g_lastLogRepeatCount = 0;
      g_lastLogMessage = "";

      long currentTf = Period();
      if(!ChartSetSymbolPeriod(0, _Symbol, (ENUM_TIMEFRAMES)currentTf))
      {
         Print("RESTART GUARD: Chart refresh failed, removing EA as fallback.");
         ExpertRemove();
      }
   }
}
//+------------------------------------------------------------------+
bool ShouldPrintOncePerWindow(string key, int secondsWindow)
{
   if(secondsWindow <= 0) return true;

   datetime now = TimeCurrent();
   for(int i = 0; i < g_logSuppressionCount; i++)
   {
      if(g_logSuppressionKeys[i] != key) continue;
      if((now - g_logSuppressionTimes[i]) < secondsWindow)
         return false;
      g_logSuppressionTimes[i] = now;
      return true;
   }

   int newSize = g_logSuppressionCount + 1;
   if(ArrayResize(g_logSuppressionKeys, newSize) != newSize) return true;
   if(ArrayResize(g_logSuppressionTimes, newSize) != newSize) return true;
   g_logSuppressionKeys[g_logSuppressionCount] = key;
   g_logSuppressionTimes[g_logSuppressionCount] = now;
   g_logSuppressionCount = newSize;
   return true;
}
//+------------------------------------------------------------------+
double GetStreakLotMultiplier()
{
   if(!INPUT_ENABLE_STREAK_LOT_MULTIPLIER || !INPUT_LOT_STREAK_BOOST_ON) return 1.0;
   if(g_streakMultiplierOrdersRemaining <= 0) return 1.0;
   if(INPUT_STREAK_LOT_MULTIPLIER <= 0) return 1.0;
   return INPUT_STREAK_LOT_MULTIPLIER;
}
//+------------------------------------------------------------------+
void ConsumeStreakMultiplierOrder()
{
   if(!INPUT_ENABLE_STREAK_LOT_MULTIPLIER || !INPUT_LOT_STREAK_BOOST_ON) return;
   if(g_streakMultiplierOrdersRemaining <= 0) return;

   g_streakMultiplierOrdersRemaining--;
   if(INPUT_ENABLE_LOGGING)
      LogWithRestartGuard("STREAK LOT BOOST USED: remainingOrders=" + IntegerToString(g_streakMultiplierOrdersRemaining));
}
//+------------------------------------------------------------------+
bool IsSpreadHigh()
{
   if(!INPUT_ENABLE_HIGH_SPREAD_PROTECT)
      return false;

   double spreadPoints = (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
   if(spreadPoints < 0) spreadPoints = 0;

   double spread = spreadPoints * g_point;
   double avgSpread = (g_averageSpread > 0) ? g_averageSpread : spread;
   if(avgSpread <= 0)
      return false;

   return (spread > avgSpread * INPUT_HIGH_SPREAD_MULTIPLIER);
}
//+------------------------------------------------------------------+
bool IsPositionProfitable(ulong ticket)
{
   if(!PositionSelectByTicket(ticket))
      return false;
   return (PositionGetDouble(POSITION_PROFIT) > 0.0);
}
//+------------------------------------------------------------------+
void HandleHighSpreadOpenPositions()
{
   if(!IsCloseEnabled() || !INPUT_CLOSE_HIGH_SPREAD_PROFIT_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effCloseHighSpreadProfit)
   {
      if(!loggedDisabled)
      {
         Print("HIGH SPREAD CLOSE disabled (INPUT_ENABLE_CLOSE_HIGH_SPREAD_PROFIT=OFF)");
         loggedDisabled = true;
      }
      return;
   }

   if(!IsSpreadHigh())
      return;
         double closePercent = MathMax(1.0, MathMin(INPUT_HIGH_SPREAD_CLOSE_PERCENT, 100.0));
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      if(!PositionSelectByTicket(ticket)) continue;

      double profit = PositionGetDouble(POSITION_PROFIT);
      if(profit > 0.0)
      {
          double currentLots = PositionGetDouble(POSITION_VOLUME);
         double step = (g_lotStep > 0.0) ? g_lotStep : g_minLot;
         if(step <= 0.0) step = 0.01;

         double lotsToClose = currentLots * (closePercent / 100.0);
         lotsToClose = MathFloor(lotsToClose / step) * step;
         lotsToClose = NormalizeDouble(lotsToClose, (int)g_lotDigits);

         // If requested percent rounds to full volume (or broker limits block partial), close full.
         bool shouldFullClose = (closePercent >= 100.0 || lotsToClose >= currentLots);

         bool closeOk = false;
         if(shouldFullClose)
            closeOk = g_trade.PositionClose(ticket);
         else if(lotsToClose >= g_minLot)
            closeOk = g_trade.PositionClosePartial(ticket, lotsToClose);

         if(closeOk && ShouldPrintOncePerWindow("high_spread_close_" + IntegerToString((int)ticket), 30))
         {
            string action = shouldFullClose ? "closed full" : ("closed " + DoubleToString(lotsToClose, (int)g_lotDigits) + " lots");
            LogWithRestartGuard("HIGH SPREAD: profitable position " + IntegerToString((int)ticket) +
                               " | " + action +
                               " | closePercent=" + DoubleToString(closePercent, 1));
         }
      }
      else if(profit <= 0.0 && INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD)
      {
         if(ShouldPrintOncePerWindow("high_spread_keep_" + IntegerToString((int)ticket), 60))
            LogWithRestartGuard("HIGH SPREAD: loss position " + IntegerToString((int)ticket) + " - keeping original stops");
      }
   }
}
//+------------------------------------------------------------------+
bool ShouldSkipStopAdjustmentsForTicket(ulong ticket)
{
   if(!INPUT_MODIFY_SUPPRESS_ON_HIGH_SPREAD_LOSS_ON)
      return false;
   if(!IsSpreadHigh())
      return false;

   if(!g_effModifySkipLossOnHighSpread || !INPUT_KEEP_LOSS_STOPS_ON_HIGH_SPREAD)
      return false;

   if(!PositionSelectByTicket(ticket))
      return false;

   return (PositionGetDouble(POSITION_PROFIT) <= 0.0);
}
//+------------------------------------------------------------------+
int GetTPFailureTrackerIndex(ulong ticket, bool createIfMissing)
{
   for(int i = 0; i < ArraySize(g_tpModifyFailures); i++)
   {
      if(g_tpModifyFailures[i].ticket == ticket)
         return i;
   }

   if(!createIfMissing)
      return -1;

   int newSize = ArraySize(g_tpModifyFailures) + 1;
   if(ArrayResize(g_tpModifyFailures, newSize) != newSize)
      return -1;

   int idx = newSize - 1;
   g_tpModifyFailures[idx].ticket = ticket;
   g_tpModifyFailures[idx].failCount = 0;
   g_tpModifyFailures[idx].nextRetryTime = 0;
   return idx;
}
//+------------------------------------------------------------------+
void ResetTPFailureTracker(ulong ticket)
{
   int idx = GetTPFailureTrackerIndex(ticket, false);
   if(idx < 0)
      return;

   g_tpModifyFailures[idx].failCount = 0;
   g_tpModifyFailures[idx].nextRetryTime = 0;
}
//+------------------------------------------------------------------+
void RegisterTPModifyFailure(ulong ticket)
{
   int idx = GetTPFailureTrackerIndex(ticket, true);
   if(idx < 0)
      return;

   g_tpModifyFailures[idx].failCount++;
   int backoffSeconds = (int)MathMin(300.0, MathPow(2.0, (double)MathMin(g_tpModifyFailures[idx].failCount, 8)));
   g_tpModifyFailures[idx].nextRetryTime = TimeCurrent() + backoffSeconds;
}
//+------------------------------------------------------------------+
bool CanAttemptTPModify(ulong ticket)
{
   int idx = GetTPFailureTrackerIndex(ticket, false);
   if(idx < 0)
      return true;

   return (TimeCurrent() >= g_tpModifyFailures[idx].nextRetryTime);
}
//+------------------------------------------------------------------+
bool IsAllowedSession()
{
   if(!INPUT_GATE_SESSION_WINDOW_ON)
      return true;

   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   int hour = dt.hour;

   if(!IsValidHourValue(hour))
   {
      WarnInvalidSessionHour("current_server_hour", hour);
      RegisterDataWarning("Invalid current session hour");
      return false;
   }

   int logicalHour = (hour - INPUT_SERVER_UTC_OFFSET_HOURS) % 24;
   if(logicalHour < 0) logicalHour += 24;

   bool inAsian  = IsHourInWindow(logicalHour, INPUT_ASIAN_START, INPUT_ASIAN_END);
   bool inLondon = IsHourInWindow(logicalHour, INPUT_LONDON_START, INPUT_LONDON_END);
   bool inNY     = IsHourInWindow(logicalHour, INPUT_NY_START, INPUT_NY_END);

   if((inAsian && inLondon) || (inAsian && inNY) || (inLondon && inNY))
      RegisterDataWarning("Overlapping session windows detected");

   bool asianOn = (INPUT_TRADE_ASIAN && INPUT_SESSION_ASIAN_ON);
   bool londonOn = (INPUT_TRADE_LONDON && INPUT_SESSION_LONDON_ON);
   bool nyOn = (INPUT_TRADE_NEWYORK && INPUT_SESSION_NY_ON);

   if(!asianOn && !londonOn && !nyOn)
      return !INPUT_SESSION_ALL_OFF_BLOCK_ENTRIES;

   if(nyOn && inNY) return true;
   if(londonOn && inLondon) return true;
   if(asianOn && inAsian) return true;

   return false;
}
//+------------------------------------------------------------------+
//+------------------------------------------------------------------+
int GetCurrentSession()
{
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   int hour = dt.hour;

   if(!IsValidHourValue(hour))
   {
      WarnInvalidSessionHour("current_server_hour", hour);
      return -1;
   }

   int logicalHour = (hour - INPUT_SERVER_UTC_OFFSET_HOURS) % 24;
   if(logicalHour < 0) logicalHour += 24;

   // Priority: NY > London > Asian (for overlaps)
   if(INPUT_SESSION_NY_ON && IsHourInWindow(logicalHour, INPUT_NY_START, INPUT_NY_END)) return 2;
   if(INPUT_SESSION_LONDON_ON && IsHourInWindow(logicalHour, INPUT_LONDON_START, INPUT_LONDON_END)) return 1;
   if(INPUT_SESSION_ASIAN_ON && IsHourInWindow(logicalHour, INPUT_ASIAN_START, INPUT_ASIAN_END)) return 0;

   return -1; // Off-hours
}
//+------------------------------------------------------------------+
//| SECTION 20: SL/TP CALCULATION (Threat?adjusted)                  |
//+------------------------------------------------------------------+
 bool CalculateSLTP(int direction, double threat, double &slPoints, double &tpPoints)
 {
    // FIXED: Use M5 ATR instead of M1 for wider, more realistic SL/TP on instruments like XAUUSD
    double atr[];
    if(CopyBuffer(g_hATR_M5, 0, 0, 1, atr) < 1 || atr[0] <= 0)
    {
       // Fallback to M1 if M5 unavailable
       if(CopyBuffer(g_hATR_M1, 0, 0, 1, atr) < 1 || atr[0] <= 0)
          return false;
    }

   // Base SL/TP from ATR
   slPoints = atr[0] * INPUT_SL_ATR_MULTIPLIER / g_point;
   tpPoints = atr[0] * INPUT_TP_ATR_MULTIPLIER / g_point;

   // Adaptive adjustments (already stored in points)
   slPoints += g_adaptive.slAdjustPoints;
   tpPoints += g_adaptive.tpAdjustPoints;

   // Sane bounds per symbol before further transforms
   double maxAdaptiveShift = MathMax(10.0, SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * 3.0);
   slPoints = MathMax(slPoints, 1.0);
   tpPoints = MathMax(tpPoints, 1.0);
   g_adaptive.slAdjustPoints = MathMax(-maxAdaptiveShift, MathMin(g_adaptive.slAdjustPoints, maxAdaptiveShift));
   g_adaptive.tpAdjustPoints = MathMax(-maxAdaptiveShift, MathMin(g_adaptive.tpAdjustPoints, maxAdaptiveShift));

   // Threat-based SL tightening
   ENUM_THREAT_ZONE zone = GetThreatZone(threat);
   switch(zone)
   {
      case THREAT_ORANGE: slPoints *= 0.80; break; // 20% tighter
      case THREAT_RED:    slPoints *= 0.60; break; // 40% tighter
      case THREAT_EXTREME:slPoints *= 0.50; break; // 50% tighter
      default: break;
   }

   // Regime-based TP adjustment
   if(g_currentRegime == REGIME_TRENDING)
      tpPoints *= 1.5; // Larger TP in trends
   else if(g_currentRegime == REGIME_RANGING)
      tpPoints *= 0.75; // Smaller TP in ranges

   double slBeforeSpread = slPoints;
   double tpBeforeSpread = tpPoints;

   // Spread-aware risk/reward shaping:
   // - Add spread to SL so effective stop includes current transaction cost.
   // - Subtract spread from TP to avoid overestimating net reachable reward.
   double spreadPoints = (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
   if(spreadPoints < 0) spreadPoints = 0;
   slPoints += spreadPoints;
   tpPoints -= spreadPoints;

   // Floor clamp before hard limits to avoid non-positive TP under wide spreads.
   tpPoints = MathMax(tpPoints, 1.0);
   slPoints = MathMax(slPoints, 1.0);

   // Re-apply user min/max limits after spread adjustment
   slPoints = MathMax(slPoints, INPUT_MIN_SL_POINTS);
   slPoints = MathMin(slPoints, INPUT_MAX_SL_POINTS);
   tpPoints = MathMax(tpPoints, INPUT_MIN_TP_POINTS);
   tpPoints = MathMin(tpPoints, INPUT_MAX_TP_POINTS);

   // Re-apply stop-level constraints after spread adjustment and clamps
   double minDist = g_stopLevel * g_point;
   if(slPoints * g_point < minDist) slPoints = (double)(g_stopLevel + 10);
   if(tpPoints * g_point < minDist) tpPoints = (double)(g_stopLevel + 10);

   if(INPUT_ENABLE_LOGGING)
   {
      Print("SLTP DEBUG: spreadPts=", DoubleToString(spreadPoints, 1),
            " | preSpread SL/TP=", DoubleToString(slBeforeSpread, 1), "/", DoubleToString(tpBeforeSpread, 1),
            " | postSpread SL/TP=", DoubleToString(slPoints, 1), "/", DoubleToString(tpPoints, 1),
            " | threat=", DoubleToString(threat, 1));
   }

   return true;
}
//+------------------------------------------------------------------+
//| SECTION 21: POSITION SIZING (Multi?factor adaptive)              |
//+------------------------------------------------------------------+
double CalculateLotSize(double slPoints, double confidence, double threat, ENUM_RL_ACTION rlAction, int direction)
{
   if(slPoints <= 0) return 0;

   double balance = AccountInfoDouble(ACCOUNT_BALANCE);
   double riskMoney = balance * (g_risk.riskPercent / 100.0);
   if(!INPUT_LOT_BASE_RISK_ON)
      riskMoney = balance * (INPUT_MIN_LOT_SIZE / MathMax(0.01, balance));

   // Base lot size from risk (using OrderCalcProfit)
   double slValue = slPoints * g_point;
   double lotSize = 0;

   double testProfit = 0;
   double price = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   if(OrderCalcProfit(ORDER_TYPE_BUY, _Symbol, 1.0, price, price + slValue, testProfit))
   {
      if(MathAbs(testProfit) > 0)
         lotSize = riskMoney / MathAbs(testProfit);
   }

   // Fallback using tick value/size
   if(!MathIsValidNumber(lotSize) || lotSize <= 0)
      lotSize = riskMoney / (slPoints * (g_tickValue / g_tickSize) * g_point);

   if(!MathIsValidNumber(lotSize) || lotSize <= 0)
      return 0;

   // Threat factor (never below 25%)
   double threatFactor = 1.0 - (threat / 200.0);
   threatFactor = MathMax(threatFactor, 0.25);
   lotSize *= threatFactor;

   // Confidence factor - less aggressive now (0.5?1.0 instead of 0?1)
   double confFactor = 0.5 + (confidence / 200.0);
   lotSize *= confFactor;

   // RL scaling
   if(INPUT_LOT_RL_SCALING_ON && INPUT_ENABLE_RL)
   {
      switch(rlAction)
      {
         case RL_HALF_SIZE:    lotSize *= 0.5; break;
         case RL_QUARTER_SIZE: lotSize *= 0.25; break;
         case RL_SKIP_TRADE:   return 0; // Should never reach here
         default: break;
      }
   }

   // Adaptive multiplier
   if(INPUT_LOT_ADAPTIVE_MULTIPLIER_ON)
      lotSize *= g_adaptive.lotMultiplier;

   // Consecutive-win lot boost (optional)
   if(INPUT_LOT_STREAK_BOOST_ON)
      lotSize *= GetStreakLotMultiplier();

   // Round to lot step and clamp to limits
   lotSize = MathFloor(lotSize / GetEffectiveLotStep()) * GetEffectiveLotStep();
   lotSize = MathMax(lotSize, g_risk.minLot);
   lotSize = MathMin(lotSize, g_risk.maxLot);
   lotSize = MathMax(lotSize, g_minLot);
   lotSize = MathMin(lotSize, g_maxLot);

   // Check margin
   double marginRequired = 0;
   ENUM_ORDER_TYPE marginOrderType = (direction >= 0 ? ORDER_TYPE_BUY : ORDER_TYPE_SELL);
   if(!OrderCalcMargin(marginOrderType, _Symbol, lotSize, price, marginRequired))
      return 0;

   double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
   if(INPUT_LOT_MARGIN_DOWNSCALE_ON && marginRequired > freeMargin * 0.8)
   {
      lotSize = (freeMargin * 0.5) / (marginRequired / lotSize);
      lotSize = MathFloor(lotSize / GetEffectiveLotStep()) * GetEffectiveLotStep();
      if(lotSize < g_minLot) return 0;
   }

   return lotSize;
}
//+------------------------------------------------------------------+
//| SECTION 22: ORDER EXECUTION                                      |
//+------------------------------------------------------------------+
bool PlacePendingStopOrder(const DecisionResult &decision, string comment, ulong &placedOrderTicket)
{
   placedOrderTicket = 0;

    int pendingSameDirection = CountMainPendingStopsByDirection(decision.direction);
   int pendingOppositeDirection = CountMainPendingStopsByDirection(decision.direction * -1);
   int openMain = CountMainPositionsFromBroker();
   int effectiveExposure = GetEffectiveMainExposureCount();

   if((INPUT_STRICT_OPPOSITE_FLIP_MODE || INPUT_MAX_MAIN_HARD_CAP_ON) && INPUT_MAX_CONCURRENT_TRADES == 1 && pendingOppositeDirection > 0)
   {
      Print("PENDING STOP REJECTED: strict one-slot cap still has opposite pending | oppositePending=", pendingOppositeDirection);
      return false;
   }

   if(INPUT_EXEC_PENDING_DUPLICATE_BLOCK_ON && pendingSameDirection > 0)
   {
       Print("PENDING STOP REJECTED: duplicate pending rule | Symbol=", _Symbol,
            " | Magic=", INPUT_MAGIC_NUMBER,
            " | Direction=", (decision.direction == 1 ? "BUY" : "SELL"),
            " | ExistingPendingSameDir=", pendingSameDirection);
      return false;
   }

   if((INPUT_STRICT_OPPOSITE_FLIP_MODE || INPUT_MAX_MAIN_HARD_CAP_ON) && effectiveExposure >= MathMax(1, INPUT_MAX_CONCURRENT_TRADES))
   {
      Print("PENDING STOP REJECTED: strict slot cap | openMain=", openMain,
            " | pendingSame=", pendingSameDirection,
            " | pendingOpposite=", pendingOppositeDirection,
            " | effectiveExposure=", effectiveExposure,
            " | cap=", MathMax(1, INPUT_MAX_CONCURRENT_TRADES));
      return false;
   }
   MqlTradeRequest request;
   MqlTradeResult result;
   ZeroMemory(request);
   ZeroMemory(result);

   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double offset = INPUT_PENDING_STOP_OFFSET_POINTS * g_point;

   request.action = TRADE_ACTION_PENDING;
   request.symbol = _Symbol;
   request.magic = BuildMagicForSubtype(SUBTYPE_MAIN);
   request.volume = decision.lotSize;
   request.deviation = 30;
   request.type_filling = g_selectedFillingMode;
   request.comment = comment;
   if(INPUT_EXEC_PENDING_EXPIRY_ON)
   {
      request.type_time = ORDER_TIME_SPECIFIED;
      request.expiration = TimeCurrent() + (INPUT_PENDING_EXPIRY_MINUTES * 60);
   }
   else
   {
      request.type_time = ORDER_TIME_GTC;
      request.expiration = 0;
   }

   if(decision.direction == 1)
   {
      request.type = ORDER_TYPE_BUY_STOP;
      request.price = NormalizeDouble(ask + offset, g_digits);
      request.sl = NormalizeDouble(request.price - decision.slPoints * g_point, g_digits);
      request.tp = NormalizeDouble(request.price + decision.tpPoints * g_point, g_digits);
   }
   else
   {
      request.type = ORDER_TYPE_SELL_STOP;
      request.price = NormalizeDouble(bid - offset, g_digits);
      request.sl = NormalizeDouble(request.price + decision.slPoints * g_point, g_digits);
      request.tp = NormalizeDouble(request.price - decision.tpPoints * g_point, g_digits);
   }
    // Trailing-TP primary mode for pending entries as well.
   if(IsTpModifyEnabled() && INPUT_MODIFY_TRAILING_TP_ON && g_effModifyTrailingTP)
      request.tp = 0.0;

   // V7.31 FIX #4: Broker distance validation for pending stop orders
   double minStopDist = g_stopLevel * g_point;
   double currentMarket = (decision.direction == 1) ? ask : bid;
   double triggerPrice = request.price;
   double slPrice = request.sl;
   double tpPrice = request.tp;
   
   // Validate: pending trigger vs current bid/ask
   double triggerDist = MathAbs(triggerPrice - currentMarket);
   if(triggerDist < minStopDist)
   {
      Print("PENDING STOP REJECTED: Trigger price too close to market | ",
            "Direction=", (decision.direction == 1 ? "BUY" : "SELL"),
            " | TriggerDist=", DoubleToString(triggerDist / g_point, 1), " pts",
            " | Required=", DoubleToString(minStopDist / g_point, 1), " pts",
            " | Market=", DoubleToString(currentMarket, g_digits),
            " | Trigger=", DoubleToString(triggerPrice, g_digits));
      return false;
   }
   
   // Validate: SL distance from trigger
   double slDist = MathAbs(slPrice - triggerPrice);
   if(slDist < minStopDist && slPrice != 0)
   {
      Print("PENDING STOP REJECTED: SL too close to trigger price | ",
            "Direction=", (decision.direction == 1 ? "BUY" : "SELL"),
            " | SLDist=", DoubleToString(slDist / g_point, 1), " pts",
            " | Required=", DoubleToString(minStopDist / g_point, 1), " pts",
            " | Trigger=", DoubleToString(triggerPrice, g_digits),
            " | SL=", DoubleToString(slPrice, g_digits));
      return false;
   }
   
   // Validate: TP distance from trigger (when TP is set)
   if(tpPrice != 0)
   {
      double tpDist = MathAbs(tpPrice - triggerPrice);
      if(tpDist < minStopDist)
      {
         Print("PENDING STOP REJECTED: TP too close to trigger price | ",
               "Direction=", (decision.direction == 1 ? "BUY" : "SELL"),
               " | TPDist=", DoubleToString(tpDist / g_point, 1), " pts",
               " | Required=", DoubleToString(minStopDist / g_point, 1), " pts",
               " | Trigger=", DoubleToString(triggerPrice, g_digits),
               " | TP=", DoubleToString(tpPrice, g_digits));
         return false;
      }
   }

   Print("PENDING STOP REQUEST: Type=", EnumToString(request.type),
         " | Trigger=", DoubleToString(request.price, g_digits),
         " | SL=", DoubleToString(request.sl, g_digits),
         " | TP=", DoubleToString(request.tp, g_digits),
         " | Expiry=", TimeToString(request.expiration, TIME_DATE|TIME_SECONDS),
         " | Lot=", DoubleToString(request.volume, 2));

   bool sent = OrderSend(request, result);
   Print("PENDING STOP RESULT: Retcode=", result.retcode,
         " | Deal=", result.deal,
         " | Order=", result.order,
         " | Comment=", result.comment);

   if(!sent || (result.retcode != TRADE_RETCODE_DONE && result.retcode != TRADE_RETCODE_PLACED))
      return false;

   placedOrderTicket = result.order;
   return true;
}
string BuildUniqueOrderComment(const string prefix, int direction)
{
   static uint seq = 0;
   seq++;
   string dirTag = (direction >= 0) ? "B" : "S";
   string comment = prefix + dirTag + "_" + IntegerToString((int)TimeCurrent()) + "_" + IntegerToString((int)GetTickCount()) + "_" + IntegerToString((int)(seq % 100000));
   if(StringLen(comment) > 31)
      comment = StringSubstr(comment, 0, 31);
   return comment;
}

ulong ResolveOpenedPositionId(ulong orderTicket, string comment)
{
   // 1) Prefer live-position lookup by our comment/magic/symbol
   int total = PositionsTotal();
   for(int i = total - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      if(PositionGetString(POSITION_COMMENT) != comment) continue;

      return ticket;
   }

   // 2) Fallback to deal history mapping (order -> position)
   if(orderTicket != 0)
   {
      if(HistorySelect(TimeCurrent() - 300, TimeCurrent() + 5))
      {
         int deals = HistoryDealsTotal();
         for(int i = deals - 1; i >= 0; i--)
         {
            ulong dealTicket = HistoryDealGetTicket(i);
            if(dealTicket == 0) continue;
            if((ulong)HistoryDealGetInteger(dealTicket, DEAL_ORDER) != orderTicket) continue;
            if(!IsOurMagic(HistoryDealGetInteger(dealTicket, DEAL_MAGIC))) continue;
            if(HistoryDealGetString(dealTicket, DEAL_SYMBOL) != _Symbol) continue;

            ulong positionId = (ulong)HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID);
            if(positionId != 0)
               return positionId;
         }
      }
   }

   return 0;
}
bool CheckTotalRiskBudgetForCandidate(int direction, double lotSize, double slPoints, string &reason)
{
   double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   if(tickSize <= 0.0 || tickValue <= 0.0)
      return true;

   double openRisk = 0.0;
   for(int i = 0; i < PositionsTotal(); i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0 || !PositionSelectByTicket(ticket) || !IsOurPosition(ticket)) continue;
      double psl = PositionGetDouble(POSITION_SL);
      double popen = PositionGetDouble(POSITION_PRICE_OPEN);
      if(psl <= 0.0 || popen <= 0.0) continue;
      double points = MathAbs(popen - psl) / g_point;
      double lot = PositionGetDouble(POSITION_VOLUME);
      openRisk += (points * g_point / tickSize) * tickValue * lot;
   }

   double candidateRisk = (slPoints * g_point / tickSize) * tickValue * lotSize;
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   if(equity <= 0.0) return true;

   double maxRiskMoney = equity * (g_risk.maxTotalRisk / 100.0);
   double totalRisk = openRisk + candidateRisk;
   if(totalRisk > maxRiskMoney)
   {
      reason = "Total risk cap exceeded basis=equity totalRisk=" + DoubleToString(totalRisk,2) +
               " cap=" + DoubleToString(maxRiskMoney,2) + " openRisk=" + DoubleToString(openRisk,2) +
               " candidateRisk=" + DoubleToString(candidateRisk,2);
      return false;
   }
   return true;
}

bool ExecuteOrder(const DecisionResult &decision)
{
   if(!IsPlacementEnabled())
   {
      if(INPUT_ENABLE_LOGGING)
         Print("EXECUTE ORDER SKIPPED: placement toggle disabled (INPUT_TOGGLE_PLACE_ORDERS=false)");
      return false;
   }

   int cooldownRemainingSec = 0;
    bool cooldownActive = IsOrderCooldownActive(cooldownRemainingSec);
   bool allowFlipCooldownBypass = (g_currentEntryIsFlip && INPUT_FLIP_BYPASS_COOLDOWN_ON);
   if(cooldownActive && !allowFlipCooldownBypass)
   {
      Print("ORDER REJECTED: Cooldown active (", cooldownRemainingSec, "s remaining)");
      g_gateDiagnostics.cooldownRejects++;
      return false;
   }
 if(cooldownActive && allowFlipCooldownBypass && !g_flipCooldownBypassLogged)
   {
      g_flipCooldownBypassLogged = true;
      Print("COOLDOWN BYPASS APPLIED: flip replacement entry allowed | remaining=", cooldownRemainingSec, "s");
   }
   string riskReject = "";
   if(!CheckTotalRiskBudgetForCandidate(decision.direction, decision.lotSize, decision.slPoints, riskReject))
   {
      Print("ORDER REJECTED: ", riskReject);
      return false;
   }
   double price, sl, tp;
   ENUM_ORDER_TYPE orderType;

   if(decision.direction == 1)
   {
      orderType = ORDER_TYPE_BUY;
      price = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
      sl = price - decision.slPoints * g_point;
      tp = price + decision.tpPoints * g_point;
   }
   else
   {
      orderType = ORDER_TYPE_SELL;
      price = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      sl = price + decision.slPoints * g_point;
      tp = price - decision.tpPoints * g_point;
   }

   sl = NormalizeDouble(sl, g_digits);
   tp = NormalizeDouble(tp, g_digits);
   price = NormalizeDouble(price, g_digits);

// Trailing-TP primary mode: open trades without static TP and let ManageTrailingTP() drive TP updates.
   if(IsTpModifyEnabled() && INPUT_MODIFY_TRAILING_TP_ON && g_effModifyTrailingTP)
      tp = 0.0;

   string comment = BuildUniqueOrderComment(COMMENT_MAIN_PREFIX, decision.direction);
    if(INPUT_EXECUTION_MODE == PENDING_STOP)
   {
      if(!IsFeatureEnabled("pending_orders"))
      {
         if(INPUT_ENABLE_LOGGING)
            Print("EXECUTE ORDER SKIPPED: pending path disabled by toggles");
         return false;
      }
      ulong pendingOrderTicket = 0;
      if(!PlacePendingStopOrder(decision, comment, pendingOrderTicket))
      {
         Print("PENDING STOP ORDER FAILED");
         return false;
      }

      Print("PENDING STOP ORDER PLACED: ", (decision.direction == 1 ? "BUY_STOP" : "SELL_STOP"),
            " | Lot: ", decision.lotSize,
            " | BasePrice: ", price,
            " | Conf: ", DoubleToString(decision.confidence, 1), "%",
            " | Threat: ", DoubleToString(decision.threatLevel, 1),
            " | Zone: ", EnumToString(decision.threatZone),
            " | Signals: ", decision.signalCombination);

      if(INPUT_ENABLE_RL && INPUT_EXEC_RECORD_RL_ON_SUBMIT && INPUT_RL_RECORD_ON)
      {
         int state = DetermineRLState(decision.confidence,
                                      decision.threatLevel,
                                      CountMainPositionsFromBroker(),
                                      CalculateDrawdownPercent(),
                                      g_consecutiveWins >= 2);
         RecordStateAction(state, decision.rlAction, pendingOrderTicket, 0,
                           (decision.direction == 1 ? SymbolInfoDouble(_Symbol, SYMBOL_ASK) + INPUT_PENDING_STOP_OFFSET_POINTS * g_point
                                                    : SymbolInfoDouble(_Symbol, SYMBOL_BID) - INPUT_PENDING_STOP_OFFSET_POINTS * g_point),
                           decision.slPoints * g_point, decision.lotSize,
                           SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE),
                           decision.confidence, decision.mtfScore,
                           GetCombinationStrengthSnapshot(decision.signalCombination));
      }

      ConsumeStreakMultiplierOrder();
      datetime orderPlacedTime = TimeCurrent();
      g_daily.pendingOrdersPlaced++;
       g_lastOrderTime = orderPlacedTime;
      if(decision.direction == 1)
         g_lastBuyOrderTime = orderPlacedTime;
      else
         g_lastSellOrderTime = orderPlacedTime;
      g_totalTrades++;
      return true;
   }

   if(!IsFeatureEnabled("market_orders"))
   {
      if(INPUT_ENABLE_LOGGING)
         Print("EXECUTE ORDER SKIPPED: market path disabled by toggles");
      return false;
   }

  int maxAttempts = INPUT_EXEC_MARKET_RETRY_ON ? 3 : 1;
   double attemptLot = decision.lotSize;
   string lotReason = "";
   double validatedLot = 0.0;
   if(!NormalizeAndValidateOrderVolume(attemptLot, validatedLot, lotReason))
   {
      Print("ORDER REJECTED BEFORE SEND: invalid initial lot | requested=", DoubleToString(attemptLot, g_lotDigits),
            " | reason=", lotReason,
            " | step=", DoubleToString(GetEffectiveLotStep(), g_lotDigits),
            " | min=", DoubleToString(g_minLot, g_lotDigits),
            " | max=", DoubleToString(g_maxLot, g_lotDigits));
      return false;
   }
   attemptLot = validatedLot;

   for(int attempt = 0; attempt < maxAttempts; attempt++)
   {
     g_trade.SetTypeFilling(g_selectedFillingMode);
      g_trade.SetExpertMagicNumber(BuildMagicForSubtype(SUBTYPE_MAIN));

      if(g_trade.PositionOpen(_Symbol, orderType, attemptLot, price, sl, tp, comment))
      {
         ulong orderTicket = g_trade.ResultOrder();
         ulong positionId = ResolveOpenedPositionId(orderTicket, comment);
         if(positionId == 0)
            positionId = orderTicket;

         Print("OPEN VALIDATION: orderTicket=", orderTicket,
               " -> positionId=", positionId);

         Print("ORDER PLACED: ", (decision.direction == 1 ? "BUY" : "SELL"),
                 " | Lot: ", attemptLot,
               " | Price: ", price,
               " | SL: ", sl, " | TP: ", tp,
               " | Conf: ", DoubleToString(decision.confidence, 1), "%",
               " | Threat: ", DoubleToString(decision.threatLevel, 1),
               " | Zone: ", EnumToString(decision.threatZone),
                             " | Signals: ", decision.signalCombination,
               " | OrderTicket: ", orderTicket,
               " | PositionId: ", positionId);

         // Track position in internal array
            DecisionResult trackedDecision = decision;
         trackedDecision.lotSize = attemptLot;

         // Track position in internal array
   TrackNewPosition(positionId, trackedDecision, comment);
         // Record RL state?action (if RL active)
         if(INPUT_ENABLE_RL && INPUT_EXEC_RECORD_RL_ON_SUBMIT && INPUT_RL_RECORD_ON)
         {
          int state = DetermineRLState(decision.confidence,
                                          decision.threatLevel,
                                          CountMainPositionsFromBroker(),
                                          CalculateDrawdownPercent(),
                                          g_consecutiveWins >= 2);
                      RecordStateAction(state, decision.rlAction, orderTicket, positionId,
                                                 price, decision.slPoints * g_point, attemptLot,
                                        SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE),
                                        decision.confidence, decision.mtfScore,
                                        GetCombinationStrengthSnapshot(decision.signalCombination));
         }

         ConsumeStreakMultiplierOrder();
         datetime orderPlacedTime = TimeCurrent();
         g_lastOrderTime = orderPlacedTime;
         if(decision.direction == 1)
            g_lastBuyOrderTime = orderPlacedTime;
         else
            g_lastSellOrderTime = orderPlacedTime;
         g_totalTrades++;

         return true;
      }

       uint retcode = g_trade.ResultRetcode();
      Print("Order attempt ", attempt + 1, " failed: ", retcode,
            " - ", g_trade.ResultComment(),
            " | requestedLot=", DoubleToString(decision.lotSize, g_lotDigits),
            " | attemptLot=", DoubleToString(attemptLot, g_lotDigits),
            " | step=", DoubleToString(GetEffectiveLotStep(), g_lotDigits),
            " | min=", DoubleToString(g_minLot, g_lotDigits),
            " | max=", DoubleToString(g_maxLot, g_lotDigits));

      if(retcode == TRADE_RETCODE_INVALID_VOLUME)
      {
         double nextLot = attemptLot - GetEffectiveLotStep();
         if(!NormalizeAndValidateOrderVolume(nextLot, validatedLot, lotReason))
         {
            Print("RETRY STOPPED: cannot normalize fallback lot after invalid volume",
                  " | previousLot=", DoubleToString(attemptLot, g_lotDigits),
                  " | nextRequested=", DoubleToString(nextLot, g_lotDigits),
                  " | reason=", lotReason);
            break;
         }

         if(validatedLot >= attemptLot)
         {
            Print("RETRY STOPPED: invalid-volume fallback did not reduce lot",
                  " | previousLot=", DoubleToString(attemptLot, g_lotDigits),
                  " | fallbackLot=", DoubleToString(validatedLot, g_lotDigits));
            break;
         }

         attemptLot = validatedLot;
      }

      // Refresh price before next attempt
      if(decision.direction == 1)
         price = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
      else
         price = SymbolInfoDouble(_Symbol, SYMBOL_BID);

   }

   Print("ORDER FAILED after ", maxAttempts, " attempts | retry=", (INPUT_EXEC_MARKET_RETRY_ON ? "ON" : "OFF"));
   return false;
}
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
void TrackNewPosition(ulong positionTicket, const DecisionResult &decision, string comment)
{
   // Clean up inactive entries first
   CleanupInactivePositions();

   if(g_positionCount >= MAX_POSITIONS)
   {
      Print("WARNING: Position tracking array full, cannot track new position");
      return;
   }

   int idx = g_positionCount;
   g_positionCount++;

   double price = 0.0;
   if(positionTicket > 0 && PositionSelectByTicket(positionTicket))
      price = PositionGetDouble(POSITION_PRICE_OPEN);

   if(price <= 0.0 || !MathIsValidNumber(price))
   {
      // Fallback only when broker position cannot be selected yet.
      price = (decision.direction == 1) ?
             SymbolInfoDouble(_Symbol, SYMBOL_ASK) :
             SymbolInfoDouble(_Symbol, SYMBOL_BID);
   }

     g_positions[idx].ticket = positionTicket;
   g_positions[idx].direction = decision.direction;
   g_positions[idx].entryPrice = price;
   g_positions[idx].slPrice = (decision.direction == 1) ?
                              price - decision.slPoints * g_point :
                              price + decision.slPoints * g_point;
   g_positions[idx].tpPrice = (decision.direction == 1) ?
                              price + decision.tpPoints * g_point :
                              price - decision.tpPoints * g_point;
   g_positions[idx].originalLots = decision.lotSize;
   g_positions[idx].currentLots = decision.lotSize;
   g_positions[idx].signalCombination = decision.signalCombination;
   g_positions[idx].comment = comment;
   g_positions[idx].entryTime = TimeCurrent();
   g_positions[idx].entrySession = GetSessionFromTime(g_positions[idx].entryTime);
   MqlDateTime entryDt;
   TimeToStruct(g_positions[idx].entryTime, entryDt);
   g_positions[idx].entryDayOfWeek = entryDt.day_of_week;
   g_positions[idx].entryRegime = g_currentRegime;
   g_positions[idx].confidenceAtEntry = decision.confidence;
   g_positions[idx].threatAtEntry = decision.threatLevel;
   g_positions[idx].mtfScoreAtEntry = decision.mtfScore;
   g_positions[idx].fingerprintId = decision.fingerprintId;
   g_positions[idx].halfSLHit = false;
   g_positions[idx].lotReduced = false;
   g_positions[idx].partialClosed = false;
    g_positions[idx].multiPartialLevel1Done = false;
   g_positions[idx].multiPartialLevel2Done = false;
   g_positions[idx].movedToBreakeven = false;
   g_positions[idx].recoveryCount = 0;
   g_positions[idx].lastRecoveryTime = 0;
   g_positions[idx].isActive = true;
   g_positions[idx].maxProfit = 0;
   g_positions[idx].maxLoss = 0;

   if(INPUT_ENABLE_LOGGING)
      Print("TRACK NEW POSITION: ticket=", positionTicket,
            " | entryPrice=", DoubleToString(g_positions[idx].entryPrice, g_digits),
            " | source=", (PositionSelectByTicket(positionTicket) ? "POSITION_PRICE_OPEN" : "snapshot"));
}
//+------------------------------------------------------------------+
//| SECTION 23: 50% LOT CLOSE SYSTEM (Part 8)                        |
//+------------------------------------------------------------------+
void Handle50PercentLotClose()
{
   // V7.33: Multi-part LOSS-based partial closing
   if(!INPUT_ENABLE_LOSS_PARTIAL_CLOSE)
      return;
      
   if(!IsCloseEnabled() || !INPUT_CLOSE_50PCT_DEFENSIVE_ON)
      return;
   
   for(int i = 0; i < g_positionCount; i++)
   {
      if(!g_positions[i].isActive) continue;
      
      // Skip recovery/aux positions
      if(StringFind(g_positions[i].comment, COMMENT_RECOVERY_PREFIX) >= 0) continue;
      if(StringFind(g_positions[i].comment, COMMENT_AVG_PREFIX) >= 0) continue;
      
      ulong ticket = g_positions[i].ticket;
      
      if(!PositionSelectByTicket(ticket))
      {
         g_positions[i].isActive = false;
         continue;
      }
      
      double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
      double slPrice = PositionGetDouble(POSITION_SL);
      double currentLots = PositionGetDouble(POSITION_VOLUME);
      int posType = (int)PositionGetInteger(POSITION_TYPE);
      
      if(slPrice == 0) continue;
      
      double slDistance = MathAbs(entryPrice - slPrice);
      if(slDistance <= 0) continue;
      
      double currentLoss = 0;
      if(posType == POSITION_TYPE_BUY)
         currentLoss = entryPrice - currentPrice;
      else
         currentLoss = currentPrice - entryPrice;
      
      if(currentLoss <= 0) continue;
      
      double lossPct = (currentLoss / slDistance) * 100.0;
      
      // V7.33: Process each loss closing level
      for(int level = 0; level < g_lossPartsCount; level++)
      {
         bool alreadyClosed = false;
         if(level == 0) alreadyClosed = g_positions[i].lossPartialLevel1Done;
         else if(level == 1) alreadyClosed = g_positions[i].lossPartialLevel2Done;
         else if(level == 2) alreadyClosed = g_positions[i].lossPartialLevel3Done;
         else if(level == 3) alreadyClosed = g_positions[i].lossPartialLevel4Done;
         
         if(alreadyClosed) continue;
         
         double triggerPct = g_lossPartTriggers[level];
         if(lossPct < triggerPct) continue;
         
         double originalLots = g_positions[i].originalLots;
         double closePercent = g_lossPartPercentages[level];
         double lotsToClose = (originalLots * closePercent) / 100.0;
         
         lotsToClose = MathFloor(lotsToClose / GetEffectiveLotStep()) * GetEffectiveLotStep();
         lotsToClose = MathMax(lotsToClose, g_minLot);
         
         if(lotsToClose >= currentLots)
         {
            lotsToClose = currentLots - g_minLot;
            if(lotsToClose < g_minLot)
            {
               if(level == 0) g_positions[i].lossPartialLevel1Done = true;
               else if(level == 1) g_positions[i].lossPartialLevel2Done = true;
               else if(level == 2) g_positions[i].lossPartialLevel3Done = true;
               else if(level == 3) g_positions[i].lossPartialLevel4Done = true;
               continue;
            }
         }
         
         if(g_trade.PositionClosePartial(ticket, lotsToClose))
         {
            double remainingLots = PositionGetDouble(POSITION_VOLUME);
            g_positions[i].currentLots = remainingLots;
            
            if(level == 0) g_positions[i].lossPartialLevel1Done = true;
            else if(level == 1) g_positions[i].lossPartialLevel2Done = true;
            else if(level == 2) g_positions[i].lossPartialLevel3Done = true;
            else if(level == 3) g_positions[i].lossPartialLevel4Done = true;
            
            if(level == 0) 
            {
               g_positions[i].lotReduced = true;
               g_positions[i].halfSLHit = true;
            }
            
            Print("✅ V7.33 LOSS PARTIAL: Ticket ", ticket,
                  " | Part ", level+1, "/", g_lossPartsCount,
                  " | Closed ", DoubleToString(lotsToClose, 2), " lots (",
                  DoubleToString(closePercent, 1), "%)",
                  " | At ", DoubleToString(lossPct, 1), "% loss",
                  " | Remaining: ", DoubleToString(remainingLots, 2));
         }
         else
         {
            Print("❌ LOSS PARTIAL FAILED: Ticket ", ticket, " | Error: ", GetLastError());
         }
      }
   }
}

//+------------------------------------------------------------------+
//| SECTION 24: PARTIAL CLOSE & TRAILING STOP                        |
//+------------------------------------------------------------------+
void ManagePartialClose()
{
   // V7.33: Multi-part PROFIT-based partial closing
   if(!INPUT_ENABLE_PROFIT_PARTIAL_CLOSE)
      return;
      
   if(!IsCloseEnabled() || !INPUT_CLOSE_PARTIAL_TP_ON)
      return;
   
   for(int i = 0; i < g_positionCount; i++)
   {
      if(!g_positions[i].isActive) continue;
      
      ulong ticket = g_positions[i].ticket;
      
      if(!PositionSelectByTicket(ticket))
      {
         g_positions[i].isActive = false;
         continue;
      }
      
      double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
      double tpPrice = PositionGetDouble(POSITION_TP);
      double currentLots = PositionGetDouble(POSITION_VOLUME);
      int posType = (int)PositionGetInteger(POSITION_TYPE);
      
      if(tpPrice == 0) continue;
      
      double tpDistance = MathAbs(tpPrice - entryPrice);
      if(tpDistance <= 0) continue;
      
      double currentProfit = 0;
      if(posType == POSITION_TYPE_BUY)
         currentProfit = currentPrice - entryPrice;
      else
         currentProfit = entryPrice - currentPrice;
      
      if(currentProfit <= 0) continue;
      
      double profitPct = (currentProfit / tpDistance) * 100.0;
      
      // V7.33: Process each profit closing level
      for(int level = 0; level < g_profitPartsCount; level++)
      {
         bool alreadyClosed = false;
         if(level == 0) alreadyClosed = g_positions[i].profitPartialLevel1Done;
         else if(level == 1) alreadyClosed = g_positions[i].profitPartialLevel2Done;
         else if(level == 2) alreadyClosed = g_positions[i].profitPartialLevel3Done;
         else if(level == 3) alreadyClosed = g_positions[i].profitPartialLevel4Done;
         
         if(alreadyClosed) continue;
         
         double triggerPct = g_profitPartTriggers[level];
         if(profitPct < triggerPct) continue;
         
         double originalLots = g_positions[i].originalLots;
         double closePercent = g_profitPartPercentages[level];
         double lotsToClose = (originalLots * closePercent) / 100.0;
         
         lotsToClose = MathFloor(lotsToClose / GetEffectiveLotStep()) * GetEffectiveLotStep();
         lotsToClose = MathMax(lotsToClose, g_minLot);
         
         if(lotsToClose >= currentLots)
         {
            lotsToClose = currentLots - g_minLot;
            if(lotsToClose < g_minLot)
            {
               if(level == 0) g_positions[i].profitPartialLevel1Done = true;
               else if(level == 1) g_positions[i].profitPartialLevel2Done = true;
               else if(level == 2) g_positions[i].profitPartialLevel3Done = true;
               else if(level == 3) g_positions[i].profitPartialLevel4Done = true;
               continue;
            }
         }
         
         if(g_trade.PositionClosePartial(ticket, lotsToClose))
         {
            double remainingLots = PositionGetDouble(POSITION_VOLUME);
            g_positions[i].currentLots = remainingLots;
            
            if(level == 0) g_positions[i].profitPartialLevel1Done = true;
            else if(level == 1) g_positions[i].profitPartialLevel2Done = true;
            else if(level == 2) g_positions[i].profitPartialLevel3Done = true;
            else if(level == 3) g_positions[i].profitPartialLevel4Done = true;
            
            if(level == 0) g_positions[i].partialClosed = true;
            
            Print("✅ V7.33 PROFIT PARTIAL: Ticket ", ticket,
                  " | Part ", level+1, "/", g_profitPartsCount,
                  " | Closed ", DoubleToString(lotsToClose, 2), " lots (",
                  DoubleToString(closePercent, 1), "%)",
                  " | At ", DoubleToString(profitPct, 1), "% profit",
                  " | Remaining: ", DoubleToString(remainingLots, 2));
            
            if(level == 0 && g_effModifyMoveToBE && !g_positions[i].movedToBreakeven)
            {
               MoveToBreakeven(ticket, entryPrice, posType);
               g_positions[i].movedToBreakeven = true;
            }
         }
         else
         {
            Print("❌ PROFIT PARTIAL FAILED: Ticket ", ticket, " | Error: ", GetLastError());
         }
      }
   }
}

//+------------------------------------------------------------------+
void MoveToBreakeven(ulong ticket, double entryPrice, int posType)
{
   if(!IsStopModifyEnabled() || !INPUT_MODIFY_BREAKEVEN_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effModifyMoveToBE)
   {
      if(!loggedDisabled)
      {
         Print("BREAKEVEN modify disabled (INPUT_ENABLE_MODIFY_MOVE_TO_BREAKEVEN=OFF)");
         loggedDisabled = true;
      }
      return;
   }

   if(ShouldSkipStopAdjustmentsForTicket(ticket)) return;
   if(!CanModifyPosition(ticket)) return;

   double currentSL = PositionGetDouble(POSITION_SL);
   double currentTP = PositionGetDouble(POSITION_TP);

   // V7.31 FIX #1: Guard against moving SL when already better than breakeven
   if(posType == POSITION_TYPE_BUY)
   {
      // For BUY: if currentSL >= entryPrice, it's already at or better than breakeven
      if(currentSL >= entryPrice && currentSL > 0)
      {
         if(INPUT_ENABLE_LOGGING)
            Print("BREAKEVEN GUARD: BUY position ticket ", ticket, 
                  " SL already at or better than breakeven | currentSL=", 
                  DoubleToString(currentSL, g_digits), 
                  " | entryPrice=", DoubleToString(entryPrice, g_digits));
         return;
      }
   }
   else // SELL position
   {
      // For SELL: if currentSL <= entryPrice and > 0, it's already at or better than breakeven
      if(currentSL <= entryPrice && currentSL > 0)
      {
         if(INPUT_ENABLE_LOGGING)
            Print("BREAKEVEN GUARD: SELL position ticket ", ticket, 
                  " SL already at or better than breakeven | currentSL=", 
                  DoubleToString(currentSL, g_digits), 
                  " | entryPrice=", DoubleToString(entryPrice, g_digits));
         return;
      }
   }

   double newSL = NormalizeDouble(entryPrice, g_digits);

   double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
   double minDist = g_stopLevel * g_point;

   if(INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON && posType == POSITION_TYPE_BUY)
   {
      if(currentPrice - newSL < minDist)
         newSL = currentPrice - minDist;
   }
   else
   {
      if(INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON && newSL - currentPrice < minDist)
         newSL = currentPrice + minDist;
   }

   if(g_trade.PositionModify(ticket, newSL, currentTP))
      Print("BREAKEVEN: Ticket ", ticket, " SL moved to ", newSL);
}
//+------------------------------------------------------------------+
void ManageTrailingStops()
{
   if(!IsStopModifyEnabled() || !INPUT_MODIFY_TRAILING_SL_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effModifyTrailingSL)
   {
      if(!loggedDisabled)
      {
         Print("TRAILING SL modify disabled (INPUT_ENABLE_MODIFY_TRAILING_SL=OFF or legacy OFF)");
         loggedDisabled = true;
      }
      return;
   }

   static datetime lastTrailCheck = 0;
   if(TimeCurrent() - lastTrailCheck < 5) return; // throttle to every 5?sec
   lastTrailCheck = TimeCurrent();

   double atr[];
   if(CopyBuffer(g_hATR_M1, 0, 0, 1, atr) < 1 || atr[0] <= 0)
      return;

   double trailDistance = atr[0] * INPUT_TRAIL_ATR_MULTIPLIER +
                          g_adaptive.trailAdjustPoints * g_point;

   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      if(ShouldSkipStopAdjustmentsForTicket(ticket)) continue;

      double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
      double currentSL = PositionGetDouble(POSITION_SL);
      double currentTP = PositionGetDouble(POSITION_TP);
      int posType = (int)PositionGetInteger(POSITION_TYPE);

      double profit = (posType == POSITION_TYPE_BUY) ?
                      currentPrice - entryPrice :
                      entryPrice - currentPrice;

      if(profit < INPUT_TRAIL_ACTIVATION_POINTS * g_point)
         continue;

      double newSL;
      if(posType == POSITION_TYPE_BUY)
         newSL = currentPrice - trailDistance;
      else
         newSL = currentPrice + trailDistance;

      newSL = NormalizeDouble(newSL, g_digits);

      double minDist = g_stopLevel * g_point;
      if(INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON && posType == POSITION_TYPE_BUY && currentPrice - newSL < minDist)
         continue;
      if(INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON && posType == POSITION_TYPE_SELL && newSL - currentPrice < minDist)
         continue;

      bool shouldMove = false;
      if(posType == POSITION_TYPE_BUY && newSL > currentSL + INPUT_TRAIL_STEP_POINTS * g_point)
         shouldMove = true;
      else if(posType == POSITION_TYPE_SELL && newSL < currentSL - INPUT_TRAIL_STEP_POINTS * g_point)
         shouldMove = true;

      if(shouldMove && g_trade.PositionModify(ticket, newSL, currentTP))
         Print("TRAILING: Ticket ", ticket, " SL moved from ", currentSL, " to ", newSL);
   }
}
//+------------------------------------------------------------------+
bool CanModifyPosition(ulong ticket)
{
   if(!PositionSelectByTicket(ticket)) return false;
   if(!INPUT_MODIFY_BROKER_DISTANCE_GUARD_ON) return true; // diagnostics bypass

   double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
   double currentSL = PositionGetDouble(POSITION_SL);
   int posType = (int)PositionGetInteger(POSITION_TYPE);

   double freezeDist = g_freezeLevel * g_point;
   if(freezeDist > 0)
   {
      double dist = MathAbs(currentPrice - currentSL);
      if(dist <= freezeDist)
         return false;
   }
   return true;
}
bool HasEnoughMargin(double lots, double price, ENUM_ORDER_TYPE orderType)
{
   double marginRequired = 0.0;
   if(!OrderCalcMargin(orderType, _Symbol, lots, price, marginRequired))
      return false;
   double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
   return (marginRequired <= freeMargin * 0.95);
}

//+------------------------------------------------------------------+
//| SECTION 25: RECOVERY MODE DISPATCHER (Part 9)                    |
//+------------------------------------------------------------------+
void MonitorRecoveryAveragingMode() { g_activeRecoveryPrefix = COMMENT_AVG_PREFIX; g_activeRecoverySubtype = SUBTYPE_AVERAGING; MonitorRecoveryAveraging(); }
void MonitorRecoveryHedgingMode()
{
   g_activeRecoveryPrefix = COMMENT_HEDGE_PREFIX;
   g_activeRecoverySubtype = SUBTYPE_RECOVERY;
   MonitorRecoveryAveraging();
}
void MonitorRecoveryGridMode()
{
   g_activeRecoveryPrefix = COMMENT_GRID_PREFIX;
   g_activeRecoverySubtype = SUBTYPE_RECOVERY;
   MonitorRecoveryAveraging();
}
void MonitorRecoveryMartingaleMode()
{
   g_activeRecoveryPrefix = COMMENT_RECOVERY_PREFIX;
   g_activeRecoverySubtype = SUBTYPE_RECOVERY;
   MonitorRecoveryAveraging();
}

//| SECTION 25: RECOVERY AVERAGING SYSTEM (Part 9)                   |
//+------------------------------------------------------------------+
void MonitorRecoveryAveraging()
{
   double threat = CalculateMarketThreat();
   ENUM_THREAT_ZONE zone = GetThreatZone(threat);

   double baseTriggerDepth = INPUT_RECOVERY_TRIGGER_DEPTH;
   double triggerDepth = baseTriggerDepth;

   if(zone == THREAT_RED)
      triggerDepth -= 10.0;
   else if(zone == THREAT_ORANGE)
      triggerDepth -= 5.0;

   if(threat >= 70.0)
      triggerDepth -= 5.0;

   triggerDepth = MathMax(5.0, MathMin(95.0, triggerDepth));

   for(int i = 0; i < g_positionCount; i++)
   {
      if(!g_positions[i].isActive) continue;
      if(g_positions[i].recoveryCount >= INPUT_MAX_RECOVERY_PER_POS) continue;

      // Skip non-main positions
      if(StringFind(g_positions[i].comment, COMMENT_RECOVERY_PREFIX) >= 0) continue;
      if(StringFind(g_positions[i].comment, COMMENT_AVG_PREFIX)      >= 0) continue;

      if(threat < INPUT_RECOVERY_THREAT_MIN) continue;

      if(g_positions[i].lastRecoveryTime > 0 &&
         TimeCurrent() - g_positions[i].lastRecoveryTime < INPUT_RECOVERY_COOLDOWN_SECONDS)
         continue;

      ulong ticket = g_positions[i].ticket;

      if(!PositionSelectByTicket(ticket))
      {
         g_positions[i].isActive = false;
         continue;
      }

      double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
      double slPrice = PositionGetDouble(POSITION_SL);
      int posType = (int)PositionGetInteger(POSITION_TYPE);

      if(slPrice == 0) continue;

      double slDist = MathAbs(entryPrice - slPrice);
      if(slDist <= 0) continue;

      double loss = 0;
      if(posType == POSITION_TYPE_BUY)
         loss = entryPrice - currentPrice;
      else
         loss = currentPrice - entryPrice;

      if(loss <= 0) continue;

      double depthPct = (loss / slDist) * 100.0;

      if(depthPct >= triggerDepth && depthPct <= 70)
      {
         double lotRatio = INPUT_RECOVERY_LOT_RATIO_MOD;
         if(threat < 50)
            lotRatio = INPUT_RECOVERY_LOT_RATIO_SAFE;
         else if(threat >= 70)
            lotRatio = INPUT_RECOVERY_LOT_RATIO_HIGH;

         if(INPUT_ENABLE_LOGGING)
         {
            Print("RECOVERY CHECK: ticket=", ticket,
                  " depthPct=", DoubleToString(depthPct, 2),
                  " triggerDepth=", DoubleToString(triggerDepth, 2),
                  " lotRatio=", DoubleToString(lotRatio, 2),
                  " threat=", DoubleToString(threat, 2));
         }

         double recLots = g_positions[i].originalLots * lotRatio;
          recLots = MathFloor(recLots / GetEffectiveLotStep()) * GetEffectiveLotStep();
         recLots = MathMax(recLots, g_minLot);
         recLots = MathMin(recLots, g_maxLot);

         PlaceRecoveryOrder(ticket, posType, recLots, slPrice, entryPrice);

         g_positions[i].recoveryCount++;
         g_positions[i].lastRecoveryTime = TimeCurrent();
      }
      else if(INPUT_ENABLE_LOGGING)
      {
         Print("RECOVERY SKIP: ticket=", ticket,
               " depthPct=", DoubleToString(depthPct, 2),
               " triggerDepth=", DoubleToString(triggerDepth, 2),
               " lotRatio=0.00");
      }
   }
}
//+------------------------------------------------------------------+
//+------------------------------------------------------------------+
bool GetRecoveryBasketMetrics(ulong parentTicket, int parentType,
                              double plannedRecoveryLots, double plannedRecoveryPrice,
                              double &combinedBreakEven, double &combinedLots)
{
   double totalLots = 0.0;
   double weightedPrice = 0.0;

   int total = PositionsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!PositionSelectByTicket(ticket) || !IsOurPosition(ticket)) continue;

      int posType = (int)PositionGetInteger(POSITION_TYPE);
      if(posType != parentType) continue;

      string comment = PositionGetString(POSITION_COMMENT);
      bool includePos = (ticket == parentTicket);
      if(!includePos)
      {
         string parentTag = IntegerToString((int)parentTicket);
         includePos = (StringFind(comment, COMMENT_AVG_PREFIX + parentTag) >= 0 ||
                       StringFind(comment, COMMENT_RECOVERY_PREFIX + parentTag) >= 0);
      }
      if(!includePos) continue;

      double lots = PositionGetDouble(POSITION_VOLUME);
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      totalLots += lots;
      weightedPrice += lots * openPrice;
   }

   totalLots += plannedRecoveryLots;
   weightedPrice += plannedRecoveryLots * plannedRecoveryPrice;

   if(totalLots <= 0.0)
      return false;

   combinedLots = totalLots;
   combinedBreakEven = weightedPrice / totalLots;
   return true;
}
//+------------------------------------------------------------------+
bool BuildValidRecoveryTP(int parentType, double price, double sl,
                          double combinedBreakEven, double &targetTP)
{
   // Target model: combined break-even + configurable profit buffer + optional risk-distance multiplier.
   // This keeps recovery exits mathematically consistent with the full basket rather than fixed heuristics.
   double minProfitBuffer = INPUT_RECOVERY_TP_BUFFER_POINTS * g_point;
   double riskDistance = MathAbs(combinedBreakEven - sl);
   double scaledDistance = riskDistance * MathMax(0.0, INPUT_RECOVERY_TP_TARGET_MULTIPLIER);
   double desiredMove = MathMax(minProfitBuffer, scaledDistance);

   double minDist = (double)MathMax(g_stopLevel, g_freezeLevel) * g_point;
   desiredMove = MathMax(desiredMove, minDist + (2.0 * g_point));

   double tp = (parentType == POSITION_TYPE_BUY) ?
               (combinedBreakEven + desiredMove) :
               (combinedBreakEven - desiredMove);

   if(parentType == POSITION_TYPE_BUY && tp <= price)
      tp = price + desiredMove;
   if(parentType == POSITION_TYPE_SELL && tp >= price)
      tp = price - desiredMove;

   targetTP = NormalizeDouble(tp, g_digits);

   if(parentType == POSITION_TYPE_BUY && (targetTP - price) < minDist)
      targetTP = NormalizeDouble(price + minDist + (2.0 * g_point), g_digits);
   if(parentType == POSITION_TYPE_SELL && (price - targetTP) < minDist)
      targetTP = NormalizeDouble(price - minDist - (2.0 * g_point), g_digits);

   return true;
}
//+------------------------------------------------------------------+
void PlaceRecoveryOrder(ulong parentTicket, int parentType, double lots,
                        double parentSL, double parentEntry)
{
   ENUM_ORDER_TYPE orderType = (parentType == POSITION_TYPE_BUY) ? ORDER_TYPE_BUY : ORDER_TYPE_SELL;
   double price = (orderType == ORDER_TYPE_BUY) ?
                  SymbolInfoDouble(_Symbol, SYMBOL_ASK) :
                  SymbolInfoDouble(_Symbol, SYMBOL_BID);

   double sl = NormalizeDouble(parentSL, g_digits);
   double combinedBE = parentEntry;
   double combinedLots = lots;
   if(!GetRecoveryBasketMetrics(parentTicket, parentType, lots, price, combinedBE, combinedLots))
   {
      Print("RECOVERY ORDER SKIPPED: unable to calculate combined basket break-even for parent ", parentTicket);
      return;
   }

   double tp = 0.0;
   if(!BuildValidRecoveryTP(parentType, price, sl, combinedBE, tp))
   {
      Print("RECOVERY ORDER SKIPPED: invalid TP model for parent ", parentTicket);
      return;
   }

   int mainCount = CountMainPositionsFromBroker();
   int recoveryCount = CountRecoveryPositions();
   if(mainCount >= INPUT_MAX_CONCURRENT_TRADES) return;
   if(recoveryCount >= INPUT_MAX_CONCURRENT_RECOVERY_TRADES) return;
   if(INPUT_GATE_DAILY_LOSS_ON && g_daily.dayStartBalance > 0.0)
   {
      double currentDailyLossPercent = (g_daily.lossToday / g_daily.dayStartBalance) * 100.0;
      if(currentDailyLossPercent >= INPUT_DAILY_LOSS_LIMIT_PERCENT) return;
   }
   if(!HasEnoughMargin(lots, price, orderType)) return;

   string comment = g_activeRecoveryPrefix + IntegerToString((int)parentTicket);

    g_trade.SetTypeFilling(g_selectedFillingMode);
   g_trade.SetExpertMagicNumber(BuildMagicForSubtype(g_activeRecoverySubtype));

   if(g_trade.PositionOpen(_Symbol, orderType, lots, price, sl, tp, comment))
   {
      Print("RECOVERY ORDER: Parent ", parentTicket,
            " | Type: ", (orderType == ORDER_TYPE_BUY ? "BUY" : "SELL"),
            " | Lots: ", DoubleToString(lots, 2),
            " | CombinedLots: ", DoubleToString(combinedLots, 2),
            " | BasketBE: ", DoubleToString(combinedBE, g_digits),
            " | SL: ", DoubleToString(sl, g_digits),
            " | TP: ", DoubleToString(tp, g_digits));
   }
   else
   {
      Print("RECOVERY ORDER FAILED: ", g_trade.ResultRetcode(),
            " - ", g_trade.ResultComment());
   }
}
//+------------------------------------------------------------------+
void CheckRecoveryTimeouts()
{
   if(!IsCloseEnabled() || !INPUT_CLOSE_RECOVERY_TIMEOUT_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effCloseRecoveryTimeout)
   {
      if(!loggedDisabled)
      {
         Print("RECOVERY TIMEOUT close disabled (INPUT_ENABLE_CLOSE_RECOVERY_TIMEOUT=OFF)");
         loggedDisabled = true;
      }
      return;
   }
   int total = PositionsTotal();
   for(int i = total - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      string comment = PositionGetString(POSITION_COMMENT);
      if(StringFind(comment, COMMENT_RECOVERY_PREFIX) < 0 &&
         StringFind(comment, COMMENT_AVG_PREFIX)      < 0)
         continue;

      datetime openTime = (datetime)PositionGetInteger(POSITION_TIME);
      int ageMinutes = (int)((TimeCurrent() - openTime) / 60);

      if(ageMinutes >= INPUT_RECOVERY_TIMEOUT_MINUTES)
      {
         if(g_trade.PositionClose(ticket))
            Print("RECOVERY TIMEOUT: Closed ticket ", ticket,
                  " after ", ageMinutes, " minutes");
      }
   }
}
//+------------------------------------------------------------------+
//| SECTION 26: POSITION SYNC & HISTORY PROCESSING                   |
//+------------------------------------------------------------------+
void SyncPositionStates()
{
   g_syncMissingCount = 0;
   g_syncNewCount = 0;
   g_syncDuplicateCount = 0;

   // 1) Mark tracked positions inactive if no longer present, and refresh live fields
   for(int i = 0; i < g_positionCount; i++)
   {
      if(!g_positions[i].isActive) continue;

      ulong ticket = g_positions[i].ticket;
      if(!PositionSelectByTicket(ticket) || !IsOurPosition(ticket))
      {
         ArchiveRecentlyClosedPositionContext(g_positions[i]);
         g_positions[i].isActive = false;
         continue;
      }

      g_positions[i].currentLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[i].slPrice = PositionGetDouble(POSITION_SL);
      g_positions[i].tpPrice = PositionGetDouble(POSITION_TP);
   }

   // 2) Track any new broker positions (e.g., pending stop activation / terminal restart / manual sync gaps)
   int total = PositionsTotal();
   for(int p = 0; p < total; p++)
   {
      ulong ticket = PositionGetTicket(p);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      int dupCount = 0;
      bool alreadyTracked = false;
      for(int i = 0; i < g_positionCount; i++)
      {
         if(g_positions[i].isActive && g_positions[i].ticket == ticket)
         {
            dupCount++;
            alreadyTracked = true;
         }
      }
      if(dupCount > 1)
      {
         g_syncDuplicateCount += (dupCount - 1);
         if(INPUT_ENABLE_LOGGING) Print("SYNC DUPLICATE: ticket tracked multiple times: ", ticket, " | dupCount=", dupCount);
      }

      if(alreadyTracked) continue;

      if(g_positionCount >= MAX_POSITIONS)
      {
         Print("WARNING: Cannot auto-track broker position, tracking array full: ", ticket);
         continue;
      }

      int idx = g_positionCount;
      g_positionCount++;

      g_positions[idx].ticket = ticket;
      g_positions[idx].entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      g_positions[idx].slPrice = PositionGetDouble(POSITION_SL);
      g_positions[idx].tpPrice = PositionGetDouble(POSITION_TP);
      g_positions[idx].originalLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[idx].currentLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[idx].direction = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
      g_positions[idx].comment = PositionGetString(POSITION_COMMENT);
      g_positions[idx].entryTime = (datetime)PositionGetInteger(POSITION_TIME);
      g_positions[idx].entrySession = GetSessionFromTime(g_positions[idx].entryTime);
      MqlDateTime syncDt1;
      TimeToStruct(g_positions[idx].entryTime, syncDt1);
      g_positions[idx].entryDayOfWeek = syncDt1.day_of_week;
      g_positions[idx].entryRegime = REGIME_UNKNOWN;
      g_positions[idx].signalCombination = "";
      g_positions[idx].fingerprintId = "";
      g_positions[idx].confidenceAtEntry = 50;
      g_positions[idx].threatAtEntry = 30;
      g_positions[idx].mtfScoreAtEntry = 0;
      g_positions[idx].halfSLHit = false;
      g_positions[idx].lotReduced = false;
      g_positions[idx].partialClosed = false;
      g_positions[idx].multiPartialLevel1Done = false;
      g_positions[idx].multiPartialLevel2Done = false;
      g_positions[idx].movedToBreakeven = false;
      g_positions[idx].recoveryCount = 0;
      g_positions[idx].lastRecoveryTime = 0;
      g_positions[idx].isActive = true;
      g_positions[idx].maxProfit = 0;
      g_positions[idx].maxLoss = 0;

      g_syncNewCount++;
      Print("SYNC NEW: Auto-tracked broker position Ticket=", ticket,
            " | Comment=", g_positions[idx].comment);
   }


   if(INPUT_ENABLE_LOGGING && (g_syncMissingCount > 0 || g_syncNewCount > 0 || g_syncDuplicateCount > 0))
      Print("SYNC SUMMARY: missing=", g_syncMissingCount, " new=", g_syncNewCount, " duplicates=", g_syncDuplicateCount,
            " | tracked=", g_positionCount);
}
//+------------------------------------------------------------------+
void SyncExistingPositions()
{
   // FIXED: Reset count first (already done in OnInit)
   g_positionCount = 0;

   int total = PositionsTotal();
   for(int i = 0; i < total && g_positionCount < MAX_POSITIONS; i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;

      int idx = g_positionCount;
      g_positionCount++;

      g_positions[idx].ticket = ticket;
      g_positions[idx].entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      g_positions[idx].slPrice = PositionGetDouble(POSITION_SL);
      g_positions[idx].tpPrice = PositionGetDouble(POSITION_TP);
      g_positions[idx].originalLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[idx].currentLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[idx].direction = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
      g_positions[idx].comment = PositionGetString(POSITION_COMMENT);
      g_positions[idx].entryTime = (datetime)PositionGetInteger(POSITION_TIME);
      g_positions[idx].entrySession = GetSessionFromTime(g_positions[idx].entryTime);
      MqlDateTime syncDt2;
      TimeToStruct(g_positions[idx].entryTime, syncDt2);
      g_positions[idx].entryDayOfWeek = syncDt2.day_of_week;
      g_positions[idx].entryRegime = REGIME_UNKNOWN;
      g_positions[idx].signalCombination = "";
      g_positions[idx].fingerprintId = "";
      g_positions[idx].confidenceAtEntry = 50;
      g_positions[idx].threatAtEntry = 30;
      g_positions[idx].mtfScoreAtEntry = 0;
      g_positions[idx].halfSLHit = false;
      g_positions[idx].lotReduced = false;
      g_positions[idx].partialClosed = false;
       g_positions[idx].multiPartialLevel1Done = false;
      g_positions[idx].multiPartialLevel2Done = false;
      g_positions[idx].movedToBreakeven = false;
      g_positions[idx].recoveryCount = 0;
      g_positions[idx].lastRecoveryTime = 0;
      g_positions[idx].isActive = true;
      g_positions[idx].maxProfit = 0;
      g_positions[idx].maxLoss = 0;

      Print("SYNCED EXISTING: Ticket ", ticket);
   }

   Print("Synced ", g_positionCount, " existing positions");
}
//+------------------------------------------------------------------+
void UpdateFingerprintOnClose(ulong positionId, double netProfit, bool isWin, datetime closeTime)
{
   if(!INPUT_ENABLE_FINGERPRINT) return;

   string fpId = "";
   string combination = "";
   int session = GetSessionFromTime(closeTime);
   MqlDateTime dt;
   TimeToStruct(closeTime, dt);
   int dayOfWeek = dt.day_of_week;
   ENUM_MARKET_REGIME regime = g_currentRegime;

   // Prefer live/tracked context first, then recently-closed archive fallback.
   PositionState ctx;
   if(FindPositionContext(positionId, ctx))
   {
      fpId = ctx.fingerprintId;
      combination = ctx.signalCombination;
   }

   // Fallback: latest training record for this position
   if(StringLen(fpId) == 0 || StringLen(combination) == 0)
   {
      for(int i = g_trainingDataCount - 1; i >= 0; i--)
      {
         if(g_trainingData[i].ticket == positionId)
         {
            if(StringLen(fpId) == 0) fpId = g_trainingData[i].fingerprintId;
            if(StringLen(combination) == 0) combination = g_trainingData[i].signalCombination;
            if(g_trainingData[i].entrySession >= 0) session = g_trainingData[i].entrySession;
            dayOfWeek = g_trainingData[i].entryDayOfWeek;
            regime = g_trainingData[i].entryRegime;
            break;
         }
      }
   }

   if(StringLen(fpId) == 0)
      fpId = combination + "_S" + IntegerToString(session) + "_D" + IntegerToString(dayOfWeek) + "_R" + IntegerToString((int)regime);

   int idx = -1;
   for(int i = 0; i < g_fingerprintCount; i++)
   {
      if(g_fingerprints[i].id == fpId)
      {
         idx = i;
         break;
      }
   }

   if(idx < 0)
   {
      if(g_fingerprintCount >= MAX_FINGERPRINTS)
      {
         if(INPUT_ENABLE_LOGGING)
            Print("FINGERPRINT UPDATE SKIPPED: storage full | id=", fpId);
         return;
      }

      idx = g_fingerprintCount;
      g_fingerprintCount++;
      ZeroMemory(g_fingerprints[idx]);
      g_fingerprints[idx].id = fpId;
      g_fingerprints[idx].signalCombination = combination;
      g_fingerprints[idx].session = session;
      g_fingerprints[idx].dayOfWeek = dayOfWeek;
      g_fingerprints[idx].regime = regime;
      g_fingerprints[idx].decayWeight = 1.0;
   }

   // Recency/decay update
   g_fingerprints[idx].decayWeight = MathMax(0.1, MathMin(1.0,
      g_fingerprints[idx].decayWeight * INPUT_LEARNING_DECAY + (1.0 - INPUT_LEARNING_DECAY)));

   g_fingerprints[idx].totalOccurrences++;
   if(isWin)
   {
      g_fingerprints[idx].wins++;
      g_fingerprints[idx].totalProfit += netProfit;
   }
   else
   {
      g_fingerprints[idx].losses++;
      g_fingerprints[idx].totalLoss += MathAbs(netProfit);
   }

   int trades = g_fingerprints[idx].wins + g_fingerprints[idx].losses;
   if(trades > 0)
      g_fingerprints[idx].winRate = (double)g_fingerprints[idx].wins / trades;

   g_fingerprints[idx].avgProfit = (g_fingerprints[idx].wins > 0) ?
      g_fingerprints[idx].totalProfit / g_fingerprints[idx].wins : 0.0;
   g_fingerprints[idx].avgLoss = (g_fingerprints[idx].losses > 0) ?
      g_fingerprints[idx].totalLoss / g_fingerprints[idx].losses : 0.0;

   g_fingerprints[idx].profitFactor = (g_fingerprints[idx].totalLoss > 0) ?
      g_fingerprints[idx].totalProfit / g_fingerprints[idx].totalLoss :
      (g_fingerprints[idx].totalProfit > 0 ? 10.0 : 0.0);

   double pfScore = MathMin(g_fingerprints[idx].profitFactor, 3.0) / 3.0; // 0..1
   g_fingerprints[idx].strengthScore = (g_fingerprints[idx].winRate * 70.0 + pfScore * 30.0) * g_fingerprints[idx].decayWeight;
   g_fingerprints[idx].strengthScore = MathMax(0.0, MathMin(100.0, g_fingerprints[idx].strengthScore));

   g_fingerprints[idx].confidenceMultiplier = 0.7 + (g_fingerprints[idx].strengthScore / 100.0) * 0.6;
   g_fingerprints[idx].confidenceMultiplier = MathMax(0.5, MathMin(1.5, g_fingerprints[idx].confidenceMultiplier));

   g_fingerprints[idx].lastSeen = closeTime;

   if(INPUT_ENABLE_LOGGING)
      Print("FINGERPRINT UPDATED: id=", g_fingerprints[idx].id,
            " | trades=", g_fingerprints[idx].totalOccurrences,
            " | winRate=", DoubleToString(g_fingerprints[idx].winRate, 2),
            " | pf=", DoubleToString(g_fingerprints[idx].profitFactor, 2),
            " | strength=", DoubleToString(g_fingerprints[idx].strengthScore, 1),
            " | mult=", DoubleToString(g_fingerprints[idx].confidenceMultiplier, 2));
}
//+------------------------------------------------------------------+
int FindPositionCloseAccumulator(ulong positionId)
{
   for(int i = 0; i < g_positionCloseAccumulatorCount; i++)
      if(g_positionCloseAccumulators[i].positionId == positionId)
         return i;
   return -1;
}

int EnsurePositionCloseAccumulator(ulong positionId)
{
   int idx = FindPositionCloseAccumulator(positionId);
   if(idx >= 0)
      return idx;

   idx = g_positionCloseAccumulatorCount;
   g_positionCloseAccumulatorCount++;
   ArrayResize(g_positionCloseAccumulators, g_positionCloseAccumulatorCount);
   ZeroMemory(g_positionCloseAccumulators[idx]);
   g_positionCloseAccumulators[idx].positionId = positionId;
   return idx;
}

void RemovePositionCloseAccumulator(int idx)
{
   if(idx < 0 || idx >= g_positionCloseAccumulatorCount)
      return;

   for(int i = idx; i < g_positionCloseAccumulatorCount - 1; i++)
      g_positionCloseAccumulators[i] = g_positionCloseAccumulators[i + 1];

   g_positionCloseAccumulatorCount--;
   ArrayResize(g_positionCloseAccumulators, g_positionCloseAccumulatorCount);
}

bool GetPositionHistoryVolumes(ulong positionId, double &openedVolume, double &closedVolume)
{
   openedVolume = 0.0;
   closedVolume = 0.0;

   if(!HistorySelectByPosition(positionId))
      return false;

   int total = HistoryDealsTotal();
   for(int i = 0; i < total; i++)
   {
      ulong deal = HistoryDealGetTicket(i);
      if(deal == 0) continue;
      if(HistoryDealGetString(deal, DEAL_SYMBOL) != _Symbol) continue;
      if(!IsOurMagic(HistoryDealGetInteger(deal, DEAL_MAGIC))) continue;

      ENUM_DEAL_ENTRY entry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(deal, DEAL_ENTRY);
      double vol = HistoryDealGetDouble(deal, DEAL_VOLUME);
      if(entry == DEAL_ENTRY_IN || entry == DEAL_ENTRY_INOUT) openedVolume += vol;
      if(entry == DEAL_ENTRY_OUT || entry == DEAL_ENTRY_INOUT) closedVolume += vol;
   }

   return true;
}

bool IsPositionTerminalClose(ulong positionId, double &remainingVolume, bool &terminalByHistory)
{
   remainingVolume = 0.0;
   terminalByHistory = false;

   bool stillOpen = PositionSelectByTicket(positionId);
   if(stillOpen)
      remainingVolume = PositionGetDouble(POSITION_VOLUME);

   double opened = 0.0;
   double closed = 0.0;
   bool historyOk = GetPositionHistoryVolumes(positionId, opened, closed);
   if(historyOk && opened > 0.0)
   {
      double epsilon = MathMax(0.0000001, g_lotStep * 0.5);
      terminalByHistory = (closed + epsilon >= opened);
   }

   return ((!stillOpen && remainingVolume <= 0.0) || terminalByHistory);
}

ENUM_MARKOV_STATE ClassifyMarkovStateFromR(double normalizedR)
{
   if(normalizedR > INPUT_MARKOV_WIN_R) return MARKOV_WIN;
   if(normalizedR < INPUT_MARKOV_LOSS_R) return MARKOV_LOSS;
   return MARKOV_EVEN;
}

void ApplyFinalClosedPositionOutcome(ulong positionId, ulong dealTicket, datetime closeTime, double finalNetProfit, bool terminalByHistory)
{
   bool isWin = (finalNetProfit > 0.0);

   if(isWin)
   {
      g_consecutiveWins++;
      if(INPUT_ENABLE_CONSEC_WIN_CONF_BOOST && g_consecutiveWins >= INPUT_CONSEC_WIN_CONF_TRIGGER)
         g_consecWinBoostTrades++;
      g_consecutiveLosses = 0;
      g_daily.winsToday++;
      g_daily.strategyWinsToday++;
      g_daily.profitToday += finalNetProfit;
      g_daily.realizedFinalPositionPnlToday += finalNetProfit;
      if(INPUT_ENABLE_STREAK_LOT_MULTIPLIER && INPUT_STREAK_TRIGGER_WINS > 0 && INPUT_STREAK_MULTIPLIER_ORDERS > 0 && g_consecutiveWins >= INPUT_STREAK_TRIGGER_WINS && dealTicket != g_lastStreakActivatedDeal)
      {
         g_streakMultiplierOrdersRemaining = INPUT_STREAK_MULTIPLIER_ORDERS;
         g_lastStreakActivatedDeal = dealTicket;
      }
   }
   else
   {
      g_consecutiveLosses++;
      g_consecutiveWins = 0;
      g_consecWinBoostTrades = 0;
      g_streakMultiplierOrdersRemaining = 0;
      g_daily.lossesToday++;
      g_daily.strategyLossesToday++;
      g_daily.lossToday += MathAbs(finalNetProfit);
      g_daily.realizedFinalPositionPnlToday += finalNetProfit;
   }

   double entryPrice = 0.0, slDistance = 0.0, lot = 0.0, tickValue = 0.0, riskBasis = 0.0, normalizedReward = finalNetProfit;
   ComputeNormalizedRLReward(positionId, finalNetProfit, normalizedReward, entryPrice, slDistance, lot, tickValue, riskBasis);

   if(INPUT_ENABLE_MARKOV && INPUT_MARKOV_UPDATE_ON)
      UpdateMarkovTransition(g_lastMarkovState, ClassifyMarkovStateFromR(normalizedReward));

   if(INPUT_ENABLE_RL)
      if(INPUT_RL_LEARNING_ON)
         UpdateRLFromTrade(positionId, INPUT_RL_USE_RAW_REWARD ? finalNetProfit : normalizedReward);

   bool allowMLRecord = INPUT_ENABLE_ML && INPUT_ML_RECORD_ON;
   bool allowComboRecord = INPUT_ENABLE_COMBINATION_ADAPTIVE && INPUT_COMBO_ADAPTIVE_RECORD_ON;
   if(allowMLRecord || allowComboRecord)
      RecordTrainingData(positionId, dealTicket, finalNetProfit, isWin);

   UpdateFingerprintOnClose(positionId, finalNetProfit, isWin, closeTime);

   Print("CLOSED POSITION FINAL: DealTicket ", dealTicket,
         " | PositionTicket: ", positionId,
         " | P&L: ", finalNetProfit,
         " | NormR: ", DoubleToString(normalizedReward, 4),
         " | TerminalByHistory: ", terminalByHistory,
         " | Win: ", isWin, " | ConsWin: ", g_consecutiveWins,
         " | ConsLoss: ", g_consecutiveLosses);
}

void ProcessClosedPositions()
{
   datetime now = TimeCurrent();
   if(g_lastHistoryProcessTime > 0 && (now - g_lastHistoryProcessTime) < INPUT_HISTORY_PROCESS_INTERVAL_SECONDS)
      return;
   g_lastHistoryProcessTime = now;

   int safetyMargin = MathMax(0, INPUT_HISTORY_SAFETY_MARGIN_SECONDS);
   datetime fromTime = (g_lastProcessedDealTime > 0) ? g_lastProcessedDealTime - safetyMargin : now - (datetime)(MathMax(1, INPUT_HISTORY_BOOTSTRAP_DAYS) * 86400);
   if(fromTime < 0) fromTime = 0;
   if(!HistorySelect(fromTime, now)) return;

   int dealsTotal = HistoryDealsTotal();
   datetime maxProcessedDealTime = g_lastProcessedDealTime;
   ulong maxProcessedDealTicketAtTime = g_lastProcessedDealTicket;

   for(int i = 0; i < dealsTotal; i++)
   {
      ulong dealTicket = HistoryDealGetTicket(i);
      if(dealTicket == 0) continue;
      if(HistoryDealGetString(dealTicket, DEAL_SYMBOL) != _Symbol) continue;
      if(!IsOurMagic(HistoryDealGetInteger(dealTicket, DEAL_MAGIC))) continue;

      ENUM_DEAL_ENTRY entry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
      if(entry != DEAL_ENTRY_OUT && entry != DEAL_ENTRY_INOUT) continue;

      datetime closeTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
      if(closeTime < g_lastProcessedDealTime) continue;
      if(closeTime == g_lastProcessedDealTime && dealTicket <= g_lastProcessedDealTicket) continue;

      ulong posId = (ulong)HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID);
      if(posId == 0) continue;

      double netProfit = HistoryDealGetDouble(dealTicket, DEAL_PROFIT) + HistoryDealGetDouble(dealTicket, DEAL_SWAP) + HistoryDealGetDouble(dealTicket, DEAL_COMMISSION);
      double dealVolume = HistoryDealGetDouble(dealTicket, DEAL_VOLUME);

      g_daily.closedDealsToday++;
      g_daily.realizedDealPnlToday += netProfit;
      g_closedDealsProcessedTotal++;
      RecordClosedDealData(dealTicket, netProfit);

      int accIdx = EnsurePositionCloseAccumulator(posId);
      g_positionCloseAccumulators[accIdx].cumulativeNetProfit += netProfit;
      g_positionCloseAccumulators[accIdx].closedVolume += dealVolume;
      g_positionCloseAccumulators[accIdx].lastDealTicket = dealTicket;
      g_positionCloseAccumulators[accIdx].lastCloseTime = closeTime;
      if(g_positionCloseAccumulators[accIdx].firstCloseTime == 0)
         g_positionCloseAccumulators[accIdx].firstCloseTime = closeTime;

      double openedVol = 0.0, closedVol = 0.0;
      if(GetPositionHistoryVolumes(posId, openedVol, closedVol))
      {
         g_positionCloseAccumulators[accIdx].openedVolume = openedVol;
         g_positionCloseAccumulators[accIdx].closedVolume = closedVol;
      }

      double remainingVolume = 0.0;
      bool terminalByHistory = false;
      bool terminal = IsPositionTerminalClose(posId, remainingVolume, terminalByHistory);
      g_positionCloseAccumulators[accIdx].terminalByHistory = terminalByHistory;

      if(!terminal)
      {
         if(INPUT_ENABLE_LOGGING)
            Print("CLOSED DEAL (PARTIAL TELEMETRY): deal=", dealTicket,
                  " | pos=", posId,
                  " | legPnL=", DoubleToString(netProfit, 2),
                  " | cumulativePnL=", DoubleToString(g_positionCloseAccumulators[accIdx].cumulativeNetProfit, 2),
                  " | remainingVol=", DoubleToString(remainingVolume, 4));
      }
      else
      {
         ApplyFinalClosedPositionOutcome(posId, dealTicket, closeTime, g_positionCloseAccumulators[accIdx].cumulativeNetProfit, g_positionCloseAccumulators[accIdx].terminalByHistory);
         RemovePositionCloseAccumulator(accIdx);
      }

      if(closeTime > maxProcessedDealTime)
      {
         maxProcessedDealTime = closeTime;
         maxProcessedDealTicketAtTime = dealTicket;
      }
      else if(closeTime == maxProcessedDealTime && dealTicket > maxProcessedDealTicketAtTime)
      {
         maxProcessedDealTicketAtTime = dealTicket;
      }
   }

   if(maxProcessedDealTime > g_lastProcessedDealTime || (maxProcessedDealTime == g_lastProcessedDealTime && maxProcessedDealTicketAtTime > g_lastProcessedDealTicket))
   {
      g_lastProcessedDealTime = maxProcessedDealTime;
      g_lastProcessedDealTicket = maxProcessedDealTicketAtTime;
   }
   else if(g_lastProcessedDealTime == 0)
   {
      g_lastProcessedDealTime = now;
      g_lastProcessedDealTicket = 0;
   }
}
//+------------------------------------------------------------------+

void UpsertTrackedPositionFromEntryDeal(ulong positionId, ulong dealTicket)
{
   if(positionId == 0 || dealTicket == 0)
      return;

   double dealPrice = HistoryDealGetDouble(dealTicket, DEAL_PRICE);
   datetime dealTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
   if(dealPrice <= 0.0 || !MathIsValidNumber(dealPrice))
      return;

   int idx = -1;
   for(int i = 0; i < g_positionCount; i++)
   {
      if(g_positions[i].isActive && g_positions[i].ticket == positionId)
      {
         idx = i;
         break;
      }
   }

   if(idx < 0)
   {
      if(g_positionCount >= MAX_POSITIONS)
         return;

      idx = g_positionCount++;
      g_positions[idx].ticket = positionId;
      g_positions[idx].comment = "";
      g_positions[idx].signalCombination = "";
      g_positions[idx].fingerprintId = "";
      g_positions[idx].confidenceAtEntry = 50.0;
      g_positions[idx].threatAtEntry = 30.0;
      g_positions[idx].mtfScoreAtEntry = 0;
      g_positions[idx].halfSLHit = false;
      g_positions[idx].lotReduced = false;
      g_positions[idx].partialClosed = false;
      g_positions[idx].multiPartialLevel1Done = false;
      g_positions[idx].multiPartialLevel2Done = false;
      g_positions[idx].movedToBreakeven = false;
      g_positions[idx].recoveryCount = 0;
      g_positions[idx].lastRecoveryTime = 0;
      g_positions[idx].maxProfit = 0;
      g_positions[idx].maxLoss = 0;
      g_positions[idx].isActive = true;
   }

   g_positions[idx].entryPrice = dealPrice;
   if(g_positions[idx].entryTime <= 0)
      g_positions[idx].entryTime = dealTime;

   if(PositionSelectByTicket(positionId))
   {
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double deltaPoints = MathAbs(openPrice - dealPrice) / g_point;
      if(deltaPoints > 3.0)
         Print("ENTRY PRICE CONSISTENCY WARNING: pos=", positionId,
               " | dealPrice=", DoubleToString(dealPrice, g_digits),
               " | positionOpen=", DoubleToString(openPrice, g_digits),
               " | deltaPts=", DoubleToString(deltaPoints, 1));
      g_positions[idx].entryPrice = openPrice;
      g_positions[idx].slPrice = PositionGetDouble(POSITION_SL);
      g_positions[idx].tpPrice = PositionGetDouble(POSITION_TP);
      g_positions[idx].currentLots = PositionGetDouble(POSITION_VOLUME);
      g_positions[idx].originalLots = MathMax(g_positions[idx].originalLots, g_positions[idx].currentLots);
      g_positions[idx].direction = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
      g_positions[idx].comment = PositionGetString(POSITION_COMMENT);
   }
}

void ProcessEntryDeals()
{
   static ulong countedPositionIds[];
   static datetime countedDayStart = 0;

   datetime now = TimeCurrent();
   MqlDateTime dt;
   TimeToStruct(now, dt);
   dt.hour = 0; dt.min = 0; dt.sec = 0;
   datetime todayStart = StructToTime(dt);
   if(countedDayStart != todayStart)
   {
      ArrayResize(countedPositionIds, 0);
      countedDayStart = todayStart;
   }

   int safetyMargin = MathMax(0, INPUT_HISTORY_SAFETY_MARGIN_SECONDS);
   datetime fromTime = (g_lastProcessedEntryDealTime > 0)
                       ? g_lastProcessedEntryDealTime - safetyMargin
                       : now - (datetime)(MathMax(1, INPUT_HISTORY_BOOTSTRAP_DAYS) * 86400);
   if(fromTime < 0) fromTime = 0;

   if(!HistorySelect(fromTime, now))
      return;

   datetime maxTime = g_lastProcessedEntryDealTime;
   ulong maxTicketAtTime = g_lastProcessedEntryDealTicket;

   for(int i = 0; i < HistoryDealsTotal(); i++)
   {
      ulong dealTicket = HistoryDealGetTicket(i);
      if(dealTicket == 0) continue;
      if(HistoryDealGetString(dealTicket, DEAL_SYMBOL) != _Symbol) continue;
      if(!IsOurMagic(HistoryDealGetInteger(dealTicket, DEAL_MAGIC))) continue;

      ENUM_DEAL_ENTRY entry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
      if(entry != DEAL_ENTRY_IN && entry != DEAL_ENTRY_INOUT)
         continue;

      datetime entryTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
      if(entryTime < g_lastProcessedEntryDealTime) continue;
      if(entryTime == g_lastProcessedEntryDealTime && dealTicket <= g_lastProcessedEntryDealTicket) continue;

      ulong orderTicket = (ulong)HistoryDealGetInteger(dealTicket, DEAL_ORDER);
      ulong positionId = (ulong)HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID);
      if(orderTicket > 0 && positionId > 0)
         RemapPendingRLToPosition(orderTicket, positionId);

      // Pending-stop fills and market entries are normalized from broker deal/position data.
      UpsertTrackedPositionFromEntryDeal(positionId, dealTicket);

      bool alreadyCounted = false;
      for(int p = 0; p < ArraySize(countedPositionIds); p++)
      {
         if(countedPositionIds[p] == positionId)
         {
            alreadyCounted = true;
            break;
         }
      }
      if(!alreadyCounted && positionId > 0)
      {
         int csz = ArraySize(countedPositionIds);
         ArrayResize(countedPositionIds, csz + 1);
         countedPositionIds[csz] = positionId;
         g_daily.tradesPlaced++;
      }
      if(INPUT_ENABLE_LOGGING)
         Print("ENTRY DEAL: deal=", dealTicket, " | order=", orderTicket,
               " | position=", positionId,
               " | uniquePositionCounted=", (!alreadyCounted ? "yes" : "no"),
               " | tradesFilledToday=", g_daily.tradesPlaced,
               " | pendingPlacedToday=", g_daily.pendingOrdersPlaced);

      if(entryTime > maxTime || (entryTime == maxTime && dealTicket > maxTicketAtTime))
      {
         maxTime = entryTime;
         maxTicketAtTime = dealTicket;
      }
   }

   if(maxTime > g_lastProcessedEntryDealTime ||
      (maxTime == g_lastProcessedEntryDealTime && maxTicketAtTime > g_lastProcessedEntryDealTicket))
   {
      g_lastProcessedEntryDealTime = maxTime;
      g_lastProcessedEntryDealTicket = maxTicketAtTime;
   }
   else if(g_lastProcessedEntryDealTime == 0)
   {
      g_lastProcessedEntryDealTime = now;
      g_lastProcessedEntryDealTicket = 0;
   }
}
//+------------------------------------------------------------------+
int GetSessionFromTime(datetime ts)
{
   MqlDateTime dt;
   TimeToStruct(ts, dt);
   int hour = dt.hour;

   if(!IsValidHourValue(hour))
   {
      WarnInvalidSessionHour("timestamp_hour", hour);
      return -1;
   }

   int logicalHour = (hour - INPUT_SERVER_UTC_OFFSET_HOURS) % 24;
   if(logicalHour < 0) logicalHour += 24;

   // Priority: NY > London > Asian (for overlaps)
   if(INPUT_SESSION_NY_ON && IsHourInWindow(logicalHour, INPUT_NY_START, INPUT_NY_END)) return 2;
   if(INPUT_SESSION_LONDON_ON && IsHourInWindow(logicalHour, INPUT_LONDON_START, INPUT_LONDON_END)) return 1;
   if(INPUT_SESSION_ASIAN_ON && IsHourInWindow(logicalHour, INPUT_ASIAN_START, INPUT_ASIAN_END)) return 0;

   return -1;
}
//+------------------------------------------------------------------+
ENUM_MARKET_REGIME EstimateRegimeAtTime(datetime ts)
{
   if(ts <= 0) return REGIME_UNKNOWN;

   int shift = iBarShift(_Symbol, PERIOD_H1, ts, true);
   if(shift < 0) shift = iBarShift(_Symbol, PERIOD_H1, ts, false);
   if(shift < 0) return REGIME_UNKNOWN;

   double adx[];
   if(CopyBuffer(g_hADX_H1, 0, shift, 1, adx) < 1) return REGIME_UNKNOWN;

   double atrNow[];
   if(CopyBuffer(g_hATR_H1, 0, shift, 1, atrNow) < 1) return REGIME_UNKNOWN;

   double atrHist[];
   int avgLen = 30;
   if(CopyBuffer(g_hATR_H1, 0, shift + 1, avgLen, atrHist) < avgLen) return REGIME_UNKNOWN;

   double atrAvg = 0.0;
   for(int i = 0; i < avgLen; i++) atrAvg += atrHist[i];
   atrAvg /= avgLen;
   if(atrAvg <= 0.0) return REGIME_UNKNOWN;

   double volRatio = atrNow[0] / atrAvg;
   if(volRatio >= 1.5) return REGIME_VOLATILE;
   if(volRatio < 0.7) return REGIME_QUIET;
   if(adx[0] >= 25.0) return REGIME_TRENDING;
   if(adx[0] < 20.0) return REGIME_RANGING;
   return REGIME_UNKNOWN;
}
//+------------------------------------------------------------------+
void RecordClosedDealData(ulong dealTicket, double netProfit)
{
   string symbol = HistoryDealGetString(dealTicket, DEAL_SYMBOL);
   long magic = HistoryDealGetInteger(dealTicket, DEAL_MAGIC);
   string filename = symbol + "_" + IntegerToString((int)magic) + "_closed_deals.csv";

   ulong posId = (ulong)HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID);
   datetime closeTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
   ENUM_DEAL_TYPE dealType = (ENUM_DEAL_TYPE)HistoryDealGetInteger(dealTicket, DEAL_TYPE);
   int direction = 0;
   string directionConfidence = "LOW";
   string signalCombination = "";
   double confidence = 0;
   double threat = 0;
   int mtfScore = 0;
   datetime sessionRefTime = closeTime;
   ENUM_MARKET_REGIME entryRegime = REGIME_UNKNOWN;

   PositionState ctx;
   if(FindPositionContext(posId, ctx))
   {
      direction = ctx.direction;
      directionConfidence = "HIGH";
      signalCombination = ctx.signalCombination;
      confidence = ctx.confidenceAtEntry;
      threat = ctx.threatAtEntry;
      mtfScore = ctx.mtfScoreAtEntry;
      sessionRefTime = ctx.entryTime;
      entryRegime = ctx.entryRegime;
   }
   else
   {
      if(HistorySelect(closeTime - 86400 * 30, closeTime + 1))
      {
         for(int i = HistoryDealsTotal() - 1; i >= 0; i--)
         {
            ulong d = HistoryDealGetTicket(i);
            if(d == 0) continue;
            if((ulong)HistoryDealGetInteger(d, DEAL_POSITION_ID) != posId) continue;
            ENUM_DEAL_ENTRY e = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(d, DEAL_ENTRY);
            if(e != DEAL_ENTRY_IN && e != DEAL_ENTRY_INOUT) continue;
            ENUM_DEAL_TYPE t = (ENUM_DEAL_TYPE)HistoryDealGetInteger(d, DEAL_TYPE);
            if(t == DEAL_TYPE_BUY) { direction = 1; directionConfidence = "MED"; break; }
            if(t == DEAL_TYPE_SELL) { direction = -1; directionConfidence = "MED"; break; }
         }
      }

      if(direction == 0)
      {
         if(dealType == DEAL_TYPE_SELL || dealType == DEAL_TYPE_BUY_CANCELED) direction = 1;
         else if(dealType == DEAL_TYPE_BUY || dealType == DEAL_TYPE_SELL_CANCELED) direction = -1;
         directionConfidence = "LOW";
      }
   }

   int session = GetSessionFromTime(sessionRefTime);
   ENUM_MARKET_REGIME closeRegimeEstimate = EstimateRegimeAtTime(closeTime);

   bool newFile = !FileIsExist(filename);
   int handle = FileOpen(filename, FILE_READ | FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;

   if(newFile)
   {
      FileWrite(handle,
                "Symbol", "Magic", "DealTicket", "PositionID", "Direction", "DirectionConfidence", "NetProfit",
                "SignalCombination", "Confidence", "Threat", "MTF", "Session", "EntryRegime", "CloseRegimeEstimate", "Timestamp");
   }

   FileSeek(handle, 0, SEEK_END);
   FileWrite(handle,
      symbol,
      magic,
      dealTicket,
      posId,
      direction,
      directionConfidence,
      netProfit,
      signalCombination,
      confidence,
      threat,
      mtfScore,
      session,
      (int)entryRegime,
      (int)closeRegimeEstimate,
      closeTime);

   FileClose(handle);
}

//+------------------------------------------------------------------+
void RecordTrainingData(ulong positionId, ulong dealTicket, double netProfit, bool isWin)
{
   // Shift array if full
   if(g_trainingDataCount >= INPUT_MAX_TRAINING_DATA)
   {
      for(int i = 0; i < INPUT_MAX_TRAINING_DATA - 1; i++)
         g_trainingData[i] = g_trainingData[i + 1];
      g_trainingDataCount = INPUT_MAX_TRAINING_DATA - 1;
   }

   int idx = g_trainingDataCount;
   g_trainingDataCount++;

   datetime closeTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
   if(closeTime <= 0) closeTime = TimeCurrent();

   g_trainingData[idx].ticket = positionId;
   g_trainingData[idx].closeTime = closeTime; // Keep closeTime from history deal
   g_trainingData[idx].profitLoss = netProfit;
   g_trainingData[idx].isWin = isWin;
   g_trainingData[idx].exitType = isWin ? "WIN_CLOSE" : "LOSS_CLOSE";
   g_trainingData[idx].closeSession = GetSessionFromTime(closeTime);

   MqlDateTime dt;
   TimeToStruct(closeTime, dt);
   g_trainingData[idx].closeDayOfWeek = dt.day_of_week;
   g_trainingData[idx].closeRegime = EstimateRegimeAtTime(closeTime);
   g_trainingData[idx].volatilityRatio = CalculateVolatilityRatio();
   g_trainingData[idx].entrySession = g_trainingData[idx].closeSession;
   g_trainingData[idx].entryDayOfWeek = g_trainingData[idx].closeDayOfWeek;
   g_trainingData[idx].entryRegime = g_trainingData[idx].closeRegime;

   // Find matching position context (live first, then recently-closed archive)
   PositionState ctx;
   if(FindPositionContext(positionId, ctx))
   {
      g_trainingData[idx].signalCombination = ctx.signalCombination;
      g_trainingData[idx].confidenceAtEntry = ctx.confidenceAtEntry;
      g_trainingData[idx].threatAtEntry = ctx.threatAtEntry;
      g_trainingData[idx].mtfScore = ctx.mtfScoreAtEntry;
      g_trainingData[idx].fingerprintId = ctx.fingerprintId;
      g_trainingData[idx].entryPrice = ctx.entryPrice;
      g_trainingData[idx].slPrice = ctx.slPrice;
      g_trainingData[idx].tpPrice = ctx.tpPrice;
      g_trainingData[idx].entryTime = ctx.entryTime;
      g_trainingData[idx].entrySession = ctx.entrySession;
      g_trainingData[idx].entryDayOfWeek = ctx.entryDayOfWeek;
      g_trainingData[idx].entryRegime = ctx.entryRegime;
      if(g_trainingData[idx].entrySession < -1 || g_trainingData[idx].entrySession > 2)
         g_trainingData[idx].entrySession = GetSessionFromTime(ctx.entryTime);
      if(g_trainingData[idx].entryDayOfWeek < 0 || g_trainingData[idx].entryDayOfWeek > 6)
      {
         MqlDateTime edt;
         TimeToStruct(ctx.entryTime, edt);
         g_trainingData[idx].entryDayOfWeek = edt.day_of_week;
      }
      g_trainingData[idx].holdingMinutes = (int)MathMax((closeTime - ctx.entryTime) / 60, 0);
   }

   // Incremental update on append; periodic full rebuild for consistency.
   UpdateCombinationStatsIncremental(g_trainingData[idx]);
   if((g_trainingDataCount % 200) == 0)
      RecalculateCombinationStats();
   else if(INPUT_ENABLE_TREE_FEATURE_MODULE && (g_trainingDataCount % 50) == 0)
      RebuildDecisionTreeFeatureModule();

   if(INPUT_ENABLE_LOGGING)
      Print("TRAINING DATA: DealTicket=", dealTicket,
            " | PositionId=", positionId,
            " | NetPnl=", netProfit,
            " | EntrySession=", g_trainingData[idx].entrySession,
            " | CloseSession=", g_trainingData[idx].closeSession);
}
//+------------------------------------------------------------------+
//| SECTION 27: ADAPTIVE OPTIMIZATION (Part 14)                      |
//+------------------------------------------------------------------+
void CheckAdaptiveOptimization()
{
   if(!INPUT_ENABLE_ADAPTIVE) return;

   int tradesSinceOpt = g_trainingDataCount - g_adaptive.tradesAtLastOpt;
   if(tradesSinceOpt < INPUT_ADAPT_INTERVAL) return;

   // Also limit to once per day
   if(g_adaptive.lastOptimization > 0 &&
      TimeCurrent() - g_adaptive.lastOptimization < 86400)
      return;

   PerformAdaptiveOptimization();
}
//+------------------------------------------------------------------+
void PerformAdaptiveOptimization()
{
   if(g_trainingDataCount < 20) return; // need enough data

   int lookback = MathMin(20, g_trainingDataCount);
   int startIdx = g_trainingDataCount - lookback;

   int wins = 0;
   double totalProfit = 0, totalLoss = 0;

   for(int i = startIdx; i < g_trainingDataCount; i++)
   {
      if(g_trainingData[i].isWin)
      {
         wins++;
         totalProfit += g_trainingData[i].profitLoss;
      }
      else
      {
         totalLoss += MathAbs(g_trainingData[i].profitLoss);
      }
   }

   double winRate = (double)wins / lookback;
   double profitFactor = (totalLoss > 0) ? totalProfit / totalLoss :
                         (totalProfit > 0 ? 10.0 : 0);

   Print("ADAPTIVE OPTIMIZATION: WR=", winRate, " PF=", profitFactor);

   const double baselineLotMultiplier = 1.0;
   const double baselineThreatMultiplier = 1.0;
   const double baselineMinConf = INPUT_MIN_CONFIDENCE;
   const int baselineMaxPositions = INPUT_MAX_CONCURRENT_TRADES;
   const double baselineSLAdjust = 0.0;
   const double baselineTPAdjust = 0.0;
   const double baselineTrailAdjust = 0.0;

   bool isUnderperform = (winRate < 0.50 || profitFactor < 1.20);
   bool isOutperform = (winRate >= 0.60 && profitFactor >= 2.0);
   bool isNeutral = (!isUnderperform && !isOutperform);

   if(isUnderperform)
   {
      // Underperforming: reduce risk moderately
      g_adaptive.lotMultiplier = g_adaptive.lotMultiplier * (1.0 - INPUT_ADAPT_UNDERPERF_LOT_REDUCE * 0.5);
      g_adaptive.slAdjustPoints -= 2.0;
      g_adaptive.threatMultiplier += 0.03;
      g_adaptive.maxPositions -= 1;
      g_adaptive.minConfThreshold += 2.0;

      Print("ADAPTIVE: Underperforming - reducing risk moderately");
   }
   else if(isNeutral)
   {
      // Neutral band: gradual reversion to baseline
      g_adaptive.lotMultiplier += (baselineLotMultiplier - g_adaptive.lotMultiplier) * 0.25;
      g_adaptive.threatMultiplier += (baselineThreatMultiplier - g_adaptive.threatMultiplier) * 0.20;
      g_adaptive.minConfThreshold += (baselineMinConf - g_adaptive.minConfThreshold) * 0.20;
      g_adaptive.slAdjustPoints += (baselineSLAdjust - g_adaptive.slAdjustPoints) * 0.25;
      g_adaptive.tpAdjustPoints += (baselineTPAdjust - g_adaptive.tpAdjustPoints) * 0.25;
      g_adaptive.trailAdjustPoints += (baselineTrailAdjust - g_adaptive.trailAdjustPoints) * 0.20;
      int posDelta = baselineMaxPositions - g_adaptive.maxPositions;
      if(posDelta != 0)
         g_adaptive.maxPositions += (posDelta > 0 ? 1 : -1);

      Print("ADAPTIVE: Neutral - reverting gradually toward baseline");
   }
   else if(isOutperform)
   {
      // Outperforming: controlled expansion + SL reversion toward neutral
      g_adaptive.trailAdjustPoints += PipsToPoints(_Symbol, INPUT_ADAPT_OVERPERF_TRAIL_ADD);
      g_adaptive.lotMultiplier += 0.04;
      if(INPUT_ALLOW_ADAPTIVE_MAX_POSITION_EXPANSION)
         g_adaptive.maxPositions += 1;
      g_adaptive.slAdjustPoints += (baselineSLAdjust - g_adaptive.slAdjustPoints) * 0.35;
      g_adaptive.minConfThreshold += (baselineMinConf - g_adaptive.minConfThreshold) * 0.15;

      Print("ADAPTIVE: Outperforming - controlled expansion enabled");
   }

   // Safety clamps for adaptive parameters
   g_adaptive.lotMultiplier = MathMax(0.50, MathMin(g_adaptive.lotMultiplier, 1.50));
   g_adaptive.slAdjustPoints = MathMax(-30.0, MathMin(g_adaptive.slAdjustPoints, 30.0));
   g_adaptive.tpAdjustPoints = MathMax(-30.0, MathMin(g_adaptive.tpAdjustPoints, 30.0));
   g_adaptive.trailAdjustPoints = MathMax(-10.0, MathMin(g_adaptive.trailAdjustPoints, 50.0));
   g_adaptive.threatMultiplier = MathMax(0.80, MathMin(g_adaptive.threatMultiplier, 1.50));
   g_adaptive.minConfThreshold = MathMax(20.0, MathMin(g_adaptive.minConfThreshold, 85.0));
  int adaptiveMaxCap = INPUT_MAX_CONCURRENT_TRADES + (INPUT_ALLOW_ADAPTIVE_MAX_POSITION_EXPANSION ? 2 : 0);
   g_adaptive.maxPositions = (int)MathMax(1, MathMin(g_adaptive.maxPositions, adaptiveMaxCap));

   if(INPUT_ENABLE_LOGGING)
      Print("ADAPTIVE BAND RESULT: under=", (isUnderperform ? "true" : "false"),
            " neutral=", (isNeutral ? "true" : "false"),
            " out=", (isOutperform ? "true" : "false"),
            " | lotMul=", DoubleToString(g_adaptive.lotMultiplier, 3),
            " | slAdj=", DoubleToString(g_adaptive.slAdjustPoints, 2),
            " | tpAdj=", DoubleToString(g_adaptive.tpAdjustPoints, 2),
            " | trailAdj=", DoubleToString(g_adaptive.trailAdjustPoints, 2),
            " | threatMul=", DoubleToString(g_adaptive.threatMultiplier, 3),
            " | minConf=", DoubleToString(g_adaptive.minConfThreshold, 2),
            " | maxPos=", g_adaptive.maxPositions);

   g_adaptive.lastOptimization = TimeCurrent();
   g_adaptive.tradesAtLastOpt = g_trainingDataCount;
}
//+------------------------------------------------------------------+
//| SECTION 28: AI INTEGRATION                                       |
//+------------------------------------------------------------------+
string JsonEscape(const string &value)
{
   string out = value;
   StringReplace(out, "\\", "\\\\");
   StringReplace(out, "\"", "\\\"");
   StringReplace(out, "\r", "\\r");
   StringReplace(out, "\n", "\\n");
   StringReplace(out, "\t", "\\t");
   return out;
}

string TruncateForLog(const string &text, int maxChars)
{
   if(maxChars <= 0 || StringLen(text) <= maxChars)
      return text;
   return StringSubstr(text, 0, maxChars) + "...";
}

bool ExtractJsonFieldString(const string &json, const string &key, string &outValue)
{
   int keyPos = StringFind(json, "\"" + key + "\"");
   if(keyPos < 0) return false;
   int colonPos = StringFind(json, ":", keyPos);
   if(colonPos < 0) return false;
   int startPos = colonPos + 1;
   while(startPos < StringLen(json) && (StringGetCharacter(json,startPos)==' ' || StringGetCharacter(json,startPos)=='\t' || StringGetCharacter(json,startPos)=='\n' || StringGetCharacter(json,startPos)=='\r')) startPos++;
   if(startPos >= StringLen(json) || StringGetCharacter(json,startPos) != '"') return false;
   startPos++;
   int endPos = startPos;
   while(endPos < StringLen(json))
   {
      if(StringGetCharacter(json,endPos) == '"' && (endPos == startPos || StringGetCharacter(json,endPos-1) != '\\')) break;
      endPos++;
   }
   if(endPos >= StringLen(json)) return false;
   outValue = StringSubstr(json, startPos, endPos - startPos);
   return true;
}

bool ExtractJsonFieldDouble(const string &json, const string &key, double &outValue)
{
   int keyPos = StringFind(json, "\"" + key + "\"");
   if(keyPos < 0) return false;
   int colonPos = StringFind(json, ":", keyPos);
   if(colonPos < 0) return false;
   int startPos = colonPos + 1;
   while(startPos < StringLen(json) && (StringGetCharacter(json,startPos)==' ' || StringGetCharacter(json,startPos)=='\t' || StringGetCharacter(json,startPos)=='\n' || StringGetCharacter(json,startPos)=='\r')) startPos++;
   int endPos = startPos;
   while(endPos < StringLen(json))
   {
      ushort ch = (ushort)StringGetCharacter(json, endPos);
      bool numeric = (ch >= '0' && ch <= '9') || ch == '.' || ch == '-' || ch == '+' || ch == 'e' || ch == 'E';
      if(!numeric) break;
      endPos++;
   }
   if(endPos <= startPos) return false;
   outValue = StringToDouble(StringSubstr(json, startPos, endPos - startPos));
   return MathIsValidNumber(outValue);
}

bool ExtractJsonFieldBool(const string &json, const string &key, bool &outValue)
{
   int keyPos = StringFind(json, "\"" + key + "\"");
   if(keyPos < 0) return false;
   int colonPos = StringFind(json, ":", keyPos);
   if(colonPos < 0) return false;
   int startPos = colonPos + 1;
   while(startPos < StringLen(json) && (StringGetCharacter(json,startPos)==' ' || StringGetCharacter(json,startPos)=='\t' || StringGetCharacter(json,startPos)=='\n' || StringGetCharacter(json,startPos)=='\r')) startPos++;
   if(startPos + 4 <= StringLen(json) && StringSubstr(json, startPos, 4) == "true") { outValue = true; return true; }
   if(startPos + 5 <= StringLen(json) && StringSubstr(json, startPos, 5) == "false") { outValue = false; return true; }
   return false;
}

bool ShouldQueryAI()
{
   if(g_aiResponse.consecutiveErrors >= 5) return false;
   if(g_aiBackoffUntil > TimeCurrent()) return false;
   if(g_lastAIQuery > 0 && TimeCurrent() - g_lastAIQuery < INPUT_AI_INTERVAL_MINUTES * 60) return false;
   return true;
}
//+------------------------------------------------------------------+
void QueryDeepSeekAI()
{
   if(StringLen(INPUT_AI_API_KEY) < 3) return;
   if(!ShouldQueryAI())
   {
      if(g_aiBackoffUntil > TimeCurrent())
         g_aiSkippedByBackoff++;
      return;
   }

   ulong requestStartMs = GetTickCount();
   g_lastAIQuery = TimeCurrent();

   double rsi[], adx[];
   CopyBuffer(g_hRSI_M1, 0, 0, 1, rsi);
   CopyBuffer(g_hADX_M1, 0, 0, 1, adx);

   string marketContext = StringFormat("Symbol: %s | RSI: %.1f | ADX: %.1f | Regime: %s | Threat: %.1f", _Symbol,
      ArraySize(rsi) > 0 ? rsi[0] : 50.0,
      ArraySize(adx) > 0 ? adx[0] : 20.0,
      EnumToString(g_currentRegime),
      CalculateMarketThreat());

   string prompt = "Analyze: " + marketContext + ". Reply JSON: {\"bias\":\"bullish/bearish/neutral\",\"confidence\":0-100,\"risk_alert\":true/false}";
   string requestBody = "{\"model\":\"deepseek-chat\",\"messages\":[{\"role\":\"user\",\"content\":\"" + JsonEscape(prompt) + "\"}],\"max_tokens\":100}";

   char post[];
   ArrayResize(post, StringLen(requestBody));
   StringToCharArray(requestBody, post);

   char result[];
   string resultHeaders;
   string headers = "Content-Type: application/json\r\nAuthorization: Bearer " + INPUT_AI_API_KEY;
   int res = WebRequest("POST", INPUT_AI_URL, headers, 3000, post, result, resultHeaders);

   g_tickMsAILastDuration = GetTickCount() - requestStartMs;

   if(res == 200)
   {
      string response = CharArrayToString(result);
      if(ParseAIResponse(response))
      {
         g_aiResponse.lastUpdate = TimeCurrent();
         g_aiResponse.consecutiveErrors = 0;
         g_aiConsecutiveTransportFailures = 0;
         g_aiBackoffUntil = 0;
         g_lastValidAIResponse = g_aiResponse;
      }
      else
      {
         g_aiResponse.consecutiveErrors++;
         g_aiResponse = g_lastValidAIResponse;
         Print("AI WARNING: Parse failed, fallback to last valid snapshot. durationMs=", g_tickMsAILastDuration);
      }
   }
   else
   {
      g_aiResponse.consecutiveErrors++;
      g_aiConsecutiveTransportFailures++;
      int backoffSeconds = (int)MathMin(300.0, MathPow(2.0, (double)MathMin(g_aiConsecutiveTransportFailures, 8)));
      g_aiBackoffUntil = TimeCurrent() + backoffSeconds;
      Print("AI Query failed: HTTP ", res,
            " | durationMs=", g_tickMsAILastDuration,
            " | backoffSec=", backoffSeconds,
            " | consecutiveTransportFailures=", g_aiConsecutiveTransportFailures);
   }
}
//+------------------------------------------------------------------+
bool ParseAIResponse(const string &response)
{
   string bias;
   double confidence = 50.0;
   bool riskAlert = false;

   bool okBias = ExtractJsonFieldString(response, "bias", bias);
   bool okConf = ExtractJsonFieldDouble(response, "confidence", confidence);
   bool okRisk = ExtractJsonFieldBool(response, "risk_alert", riskAlert);

   if(!okBias || !okConf || !okRisk || !MathIsValidNumber(confidence) || confidence < 0.0 || confidence > 100.0)
   {
      Print("AI PARSE FAILURE: missing/invalid fields | raw=", TruncateForLog(response, 300));
      return false;
   }

   string normalizedBias = bias;
   StringToLower(normalizedBias);
   if(normalizedBias != "bullish" && normalizedBias != "bearish" && normalizedBias != "neutral")
      normalizedBias = "neutral";

   g_aiResponse.marketBias = normalizedBias;
   g_aiResponse.confidenceScore = confidence;
   g_aiResponse.riskAlert = riskAlert;
   return true;
}
//+------------------------------------------------------------------+
//| SECTION 29: DAILY RESET & UTILITIES                              |
//+------------------------------------------------------------------+
void ResetDailyCounters()
{
   g_daily.dayStart = iTime(_Symbol, PERIOD_D1, 0);
   g_daily.dayStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   g_daily.tradesPlaced = 0;
   g_daily.pendingOrdersPlaced = 0;
   g_daily.closedDealsToday = 0;
   g_daily.winsToday = 0;
   g_daily.lossesToday = 0;
   g_daily.profitToday = 0;
   g_daily.lossToday = 0;
   g_daily.peakEquityToday = AccountInfoDouble(ACCOUNT_EQUITY);
   g_daily.realizedDealPnlToday = 0.0;
   g_daily.realizedFinalPositionPnlToday = 0.0;
   g_daily.strategyWinsToday = 0;
   g_daily.strategyLossesToday = 0;
}
void CheckDailyReset()
{
   datetime currentDayStart = iTime(_Symbol, PERIOD_D1, 0);
   if(currentDayStart != g_daily.dayStart)
   {
      if(INPUT_RESET_CONSEC_DAILY)
         g_consecutiveLosses = 0;

      ResetDailyCounters();
      g_peakEquity = AccountInfoDouble(ACCOUNT_EQUITY);

      CleanupInactivePositions();

      Print("=== NEW DAY RESET === | Daily loss baseline balance=", DoubleToString(g_daily.dayStartBalance, 2));
   }
}
//+------------------------------------------------------------------+
void UpdateAverageSpread()
{
   double currentSpread = (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) * g_point;
   g_totalSpread += currentSpread;
   g_spreadSamples++;
   g_averageSpread = g_totalSpread / g_spreadSamples;

   if(g_spreadSamples > 10000)
   {
      g_totalSpread = g_averageSpread * 5000;
      g_spreadSamples = 5000;
   }
}
//+------------------------------------------------------------------+
void CloseAllPositions(string reason)
{
   if(!IsCloseEnabled())
      return;
   if(!g_effCloseAllApi)
   {
      if(ShouldPrintOncePerWindow("close_all_api_disabled", 60))
         Print("CLOSE ALL skipped (CLOSE_ALL_API_DISABLED) | reason=", reason);
      return;
   }

   bool safeOnlyOur = true;
   bool safeSymbolCurrent = true;
   if(!g_effCloseAllOnlyOur || !(g_effCloseAllSymbolFilter && INPUT_CLOSE_ALL_SYMBOL_SCOPE_CURRENT))
      Print("WARNING: CloseAllPositions broad scope request detected but safety policy forces only-our + current-symbol.");

   int attempted = 0;
   int closed = 0;
   int failed = 0;
   int total = PositionsTotal();
   for(int i = total - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!PositionSelectByTicket(ticket)) continue;

      if(safeOnlyOur && !IsOurPosition(ticket))
         continue;

      if(safeSymbolCurrent)
      {
         if(PositionGetString(POSITION_SYMBOL) != _Symbol)
            continue;
      }

      attempted++;
      if(g_trade.PositionClose(ticket))
      {
         closed++;
         Print("CLOSED: Ticket ", ticket, " | Reason: ", reason);
      }
      else
      {
         failed++;
      }
   }

   Print("CLOSE_ALL SUMMARY: reason=", reason,
         " attempted=", attempted,
         " closed=", closed,
         " failed=", failed,
         " onlyOur=", (safeOnlyOur ? "ON" : "OFF"),
         " symbolFilter=", (safeSymbolCurrent ? "ON" : "OFF"));
}
//+------------------------------------------------------------------+
void ArchiveRecentlyClosedPositionContext(const PositionState &state)
{
   if(state.ticket == 0)
      return;

   int cap = ArraySize(g_recentlyClosedContext);
   if(cap <= 0)
   {
      cap = 256;
      ArrayResize(g_recentlyClosedContext, cap);
   }

   if(g_recentlyClosedContextCount < cap)
   {
      g_recentlyClosedContext[g_recentlyClosedContextCount].state = state;
      g_recentlyClosedContext[g_recentlyClosedContextCount].archivedAt = TimeCurrent();
      g_recentlyClosedContextCount++;
      return;
   }

   for(int i = 1; i < cap; i++)
      g_recentlyClosedContext[i - 1] = g_recentlyClosedContext[i];

   g_recentlyClosedContext[cap - 1].state = state;
   g_recentlyClosedContext[cap - 1].archivedAt = TimeCurrent();
}
//+------------------------------------------------------------------+
bool FindPositionContext(ulong positionId, PositionState &ctx)
{
   for(int i = 0; i < g_positionCount; i++)
   {
      if(g_positions[i].ticket != positionId)
         continue;
      ctx = g_positions[i];
      return true;
   }

   for(int i = g_recentlyClosedContextCount - 1; i >= 0; i--)
   {
      if(g_recentlyClosedContext[i].state.ticket != positionId)
         continue;
      ctx = g_recentlyClosedContext[i].state;
      return true;
   }

   return false;
}
//+------------------------------------------------------------------+
void CleanupRecentClosedContext()
{
   if(g_recentlyClosedContextCount <= 0)
      return;

   int maxAgeHours = MathMax(24, INPUT_RL_PENDING_MAX_AGE_HOURS);
   int maxAgeSec = maxAgeHours * 3600;
   datetime now = TimeCurrent();

   int writeIdx = 0;
   for(int i = 0; i < g_recentlyClosedContextCount; i++)
   {
      bool stale = (maxAgeSec > 0 && (now - g_recentlyClosedContext[i].archivedAt) > maxAgeSec);
      if(stale) continue;
      if(writeIdx != i)
         g_recentlyClosedContext[writeIdx] = g_recentlyClosedContext[i];
      writeIdx++;
   }
   g_recentlyClosedContextCount = writeIdx;
}
//+------------------------------------------------------------------+
void CleanupInactivePositions()
{
   int writeIdx = 0;
   for(int i = 0; i < g_positionCount; i++)
   {
      if(g_positions[i].isActive)
      {
         if(writeIdx != i) g_positions[writeIdx] = g_positions[i];
         writeIdx++;
      }
   }
   g_positionCount = writeIdx;
}
//+------------------------------------------------------------------+
//| SECTION 30: DATA PERSISTENCE                                     |
//+------------------------------------------------------------------+
void SaveQTable()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_qtable.bin";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   uint checksum = FNV1aStart();
   checksum = FNV1aUpdateInt(checksum, QTABLE_HASH_SENTINEL);
   checksum = FNV1aUpdateInt(checksum, QTABLE_SCHEMA_VERSION);
   checksum = FNV1aUpdateInt(checksum, Q_TABLE_STATES);
   checksum = FNV1aUpdateInt(checksum, Q_TABLE_ACTIONS);
   checksum = FNV1aUpdateInt(checksum, g_rlTradesCompleted);

   FileWriteInteger(handle, QTABLE_SCHEMA_VERSION);
   FileWriteInteger(handle, g_rlTradesCompleted);

   for(int s = 0; s < Q_TABLE_STATES; s++)
      for(int a = 0; a < Q_TABLE_ACTIONS; a++)
      {
         FileWriteDouble(handle, g_qTable[s][a]);
         checksum = FNV1aUpdateDouble(checksum, g_qTable[s][a]);
      }

   for(int s = 0; s < Q_TABLE_STATES; s++)
      for(int a = 0; a < Q_TABLE_ACTIONS; a++)
      {
         FileWriteInteger(handle, g_qVisits[s][a]);
         checksum = FNV1aUpdateInt(checksum, g_qVisits[s][a]);
      }

   FileWriteInteger(handle, (int)checksum);
   FileFlush(handle);
   FileClose(handle);

   if(!ReplaceFileAtomic(tmpName, filename))
      Print("ERROR: SaveRuntimeState replace failed, previous runtime file kept intact: ", filename);
   Print("Q-Table saved: ", g_rlTradesCompleted, " trades recorded");
}
//+------------------------------------------------------------------+
void LoadQTable()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_qtable.bin";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   int version = FileReadInteger(handle);
   if(version != 2 && version != QTABLE_SCHEMA_VERSION)
   {
      Print("WARNING: Q-Table schema mismatch. Resetting.");
      FileClose(handle);
      ArrayInitialize(g_qTable, 0);
      ArrayInitialize(g_qVisits, 0);
      g_rlTradesCompleted = 0;
      return;
   }

   if(version == 2)
   {
      int checksumLegacy = FileReadInteger(handle);
      g_rlTradesCompleted = FileReadInteger(handle);
      for(int s = 0; s < Q_TABLE_STATES; s++) for(int a = 0; a < Q_TABLE_ACTIONS; a++) g_qTable[s][a] = FileReadDouble(handle);
      for(int s = 0; s < Q_TABLE_STATES; s++) for(int a = 0; a < Q_TABLE_ACTIONS; a++) g_qVisits[s][a] = FileReadInteger(handle);
      int verifySum = 0;
      for(int s = 0; s < Q_TABLE_STATES; s++) for(int a = 0; a < Q_TABLE_ACTIONS; a++) verifySum += (int)(g_qTable[s][a] * 100);
      if(verifySum != checksumLegacy) { ArrayInitialize(g_qTable, 0); ArrayInitialize(g_qVisits, 0); g_rlTradesCompleted = 0; }
      FileClose(handle);
      return;
   }

   g_rlTradesCompleted = FileReadInteger(handle);
   uint checksum = FNV1aStart();
   checksum = FNV1aUpdateInt(checksum, QTABLE_HASH_SENTINEL);
   checksum = FNV1aUpdateInt(checksum, QTABLE_SCHEMA_VERSION);
   checksum = FNV1aUpdateInt(checksum, Q_TABLE_STATES);
   checksum = FNV1aUpdateInt(checksum, Q_TABLE_ACTIONS);
   checksum = FNV1aUpdateInt(checksum, g_rlTradesCompleted);

   for(int s = 0; s < Q_TABLE_STATES; s++)
      for(int a = 0; a < Q_TABLE_ACTIONS; a++)
      {
         g_qTable[s][a] = FileReadDouble(handle);
         checksum = FNV1aUpdateDouble(checksum, g_qTable[s][a]);
      }

   for(int s = 0; s < Q_TABLE_STATES; s++)
      for(int a = 0; a < Q_TABLE_ACTIONS; a++)
      {
         g_qVisits[s][a] = FileReadInteger(handle);
         checksum = FNV1aUpdateInt(checksum, g_qVisits[s][a]);
      }

   int fileChecksum = FileReadInteger(handle);
   FileClose(handle);
   if((int)checksum != fileChecksum)
   {
      Print("WARNING: Q-Table checksum mismatch. Resetting.");
      ArrayInitialize(g_qTable, 0);
      ArrayInitialize(g_qVisits, 0);
      g_rlTradesCompleted = 0;
      return;
   }
   Print("Q-Table loaded: ", g_rlTradesCompleted, " trades");
}

void SaveMarkovData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_markov.bin";
   int handle = FileOpen(filename, FILE_WRITE | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   FileWriteInteger(handle, 5);
   FileWriteInteger(handle, g_markovTradesRecorded);
   FileWriteInteger(handle, (int)g_lastMarkovState);
   FileWriteInteger(handle, g_markovQueueHead);
   FileWriteInteger(handle, g_markovQueueCount);

   for(int i = 0; i < MARKOV_STATES; i++)
      for(int j = 0; j < MARKOV_STATES; j++)
         FileWriteInteger(handle, g_markovCounts[i][j]);

   int cap = ArraySize(g_markovQueue);
   FileWriteInteger(handle, cap);
   for(int i = 0; i < cap; i++)
   {
      FileWriteInteger(handle, (int)g_markovQueue[i].fromState);
      FileWriteInteger(handle, (int)g_markovQueue[i].toState);
      FileWriteLong(handle, (long)g_markovQueue[i].observedAt);
   }

   FileClose(handle);
}
//+------------------------------------------------------------------+
void LoadMarkovData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_markov.bin";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   int version = FileReadInteger(handle);
   if(version < 2 || version > 5)
   {
      Print("WARNING: Markov schema mismatch. Keeping defaults.");
      FileClose(handle);
      return;
   }

   g_markovTradesRecorded = FileReadInteger(handle);
   g_lastMarkovState = (ENUM_MARKOV_STATE)FileReadInteger(handle);
   g_markovQueueHead = (version >= 5) ? FileReadInteger(handle) : 0;
   g_markovQueueCount = (version >= 3) ? FileReadInteger(handle) : 0;

   for(int i = 0; i < MARKOV_STATES; i++)
      for(int j = 0; j < MARKOV_STATES; j++)
         g_markovCounts[i][j] = FileReadInteger(handle);

   int cap = (version >= 5) ? FileReadInteger(handle) : g_markovQueueCount;
   if(cap < 0) cap = 0;
   ArrayResize(g_markovQueue, cap);
   if(g_markovQueueCount > cap) g_markovQueueCount = cap;

   for(int i = 0; i < cap; i++)
   {
      g_markovQueue[i].fromState = (ENUM_MARKOV_STATE)FileReadInteger(handle);
      g_markovQueue[i].toState = (ENUM_MARKOV_STATE)FileReadInteger(handle);
      g_markovQueue[i].observedAt = (version >= 5) ? (datetime)FileReadLong(handle) : 0;
   }

   if(cap > 0) g_markovQueueHead = ((g_markovQueueHead % cap) + cap) % cap;
   else g_markovQueueHead = 0;

   RecomputeMarkovTransitionsFromCounts();
   FileClose(handle);
}
//+------------------------------------------------------------------+
void SaveFingerprintData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_fp.csv";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;

   FileWrite(handle, "Schema", FP_SCHEMA_VERSION, "Rows", g_fingerprintCount);
   FileWrite(handle, "ID", "Combo", "Session", "Day", "Regime", "Total", "Wins", "Losses",
             "WinRate", "PF", "AvgProfit", "AvgLoss", "Strength", "Multiplier", "Decay");

   for(int i = 0; i < g_fingerprintCount; i++)
   {
      FileWrite(handle,
         g_fingerprints[i].id,
         g_fingerprints[i].signalCombination,
         g_fingerprints[i].session,
         g_fingerprints[i].dayOfWeek,
         (int)g_fingerprints[i].regime,
         g_fingerprints[i].totalOccurrences,
         g_fingerprints[i].wins,
         g_fingerprints[i].losses,
         g_fingerprints[i].winRate,
         g_fingerprints[i].profitFactor,
         g_fingerprints[i].avgProfit,
         g_fingerprints[i].avgLoss,
         g_fingerprints[i].strengthScore,
         g_fingerprints[i].confidenceMultiplier,
         g_fingerprints[i].decayWeight);
   }

   FileClose(handle);
   if(!ReplaceFileAtomic(tmpName, filename))
      Print("WARNING: Failed atomic replace for fingerprint file ", filename);
}
//+------------------------------------------------------------------+
void LoadFingerprintData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_fp.csv";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;
   if(FileSize(handle) <= 0)
   {
      RegisterDataWarning("Zero-length persistence file: " + filename);
      FileClose(handle);
      return;
   }

   // Metadata line (schema + row count)
   string schemaKey = FileReadString(handle);
   int schemaVer = (int)FileReadNumber(handle);
   string rowsKey = FileReadString(handle);
   int rowCount = (int)FileReadNumber(handle);
   if(schemaKey != "Schema" || rowsKey != "Rows")
   {
      Print("WARNING: Fingerprint CSV header corrupt. Skipping load.");
      FileClose(handle);
      return;
   }

   for(int h = 0; h < 15; h++)
      FileReadString(handle);

   g_fingerprintCount = 0;
   int badRows = 0;
   while(!FileIsEnding(handle) && g_fingerprintCount < MAX_FINGERPRINTS)
   {
      SignalFingerprint row;
      row.id = FileReadString(handle);
      row.signalCombination = FileReadString(handle);
      row.session = (int)FileReadNumber(handle);
      row.dayOfWeek = (int)FileReadNumber(handle);
      row.regime = (ENUM_MARKET_REGIME)(int)FileReadNumber(handle);
      row.totalOccurrences = (int)FileReadNumber(handle);
      row.wins = (int)FileReadNumber(handle);
      row.losses = (int)FileReadNumber(handle);
      row.winRate = FileReadNumber(handle);
      row.profitFactor = FileReadNumber(handle);
      row.avgProfit = FileReadNumber(handle);
      row.avgLoss = FileReadNumber(handle);
      row.strengthScore = FileReadNumber(handle);
      row.confidenceMultiplier = FileReadNumber(handle);
      row.decayWeight = FileReadNumber(handle);

      bool valid = (StringLen(row.id) > 0 && row.session >= -1 && row.session <= 2 &&
                    row.dayOfWeek >= 0 && row.dayOfWeek <= 6 &&
                    MathIsValidNumber(row.winRate) && row.winRate >= 0.0 && row.winRate <= 100.0 &&
                    MathIsValidNumber(row.confidenceMultiplier) && row.confidenceMultiplier >= 0.1 && row.confidenceMultiplier <= 3.0);
      if(!valid)
      {
         badRows++;
         continue;
      }

      g_fingerprints[g_fingerprintCount++] = row;
   }

   FileClose(handle);
   if(schemaVer != FP_SCHEMA_VERSION)
      Print("WARNING: Fingerprint schema mismatch file=", schemaVer, " expected=", FP_SCHEMA_VERSION);
   Print("Fingerprints loaded: ", g_fingerprintCount, " | badRows=", badRows, " | declaredRows=", rowCount);
}
//+------------------------------------------------------------------+
void SaveTrainingData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_training.csv";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;

   FileWrite(handle, "Schema", TRAINING_SCHEMA_VERSION, "Rows", g_trainingDataCount);
   FileWrite(handle, "Ticket", "EntryTime", "CloseTime", "Combo", "Profit", "IsWin",
             "Confidence", "Threat", "MTF", "VolRatio", "EntrySession", "CloseSession",
             "EntryDay", "CloseDay", "EntryRegime", "CloseRegime", "FP");

   for(int i = 0; i < g_trainingDataCount; i++)
   {
      FileWrite(handle,
         g_trainingData[i].ticket,
         g_trainingData[i].entryTime,
         g_trainingData[i].closeTime,
         g_trainingData[i].signalCombination,
         g_trainingData[i].profitLoss,
         g_trainingData[i].isWin ? 1 : 0,
         g_trainingData[i].confidenceAtEntry,
         g_trainingData[i].threatAtEntry,
         g_trainingData[i].mtfScore,
         g_trainingData[i].volatilityRatio,
         g_trainingData[i].entrySession,
         g_trainingData[i].closeSession,
         g_trainingData[i].entryDayOfWeek,
         g_trainingData[i].closeDayOfWeek,
         (int)g_trainingData[i].entryRegime,
         (int)g_trainingData[i].closeRegime,
         g_trainingData[i].fingerprintId);
   }

   FileClose(handle);
   if(!ReplaceFileAtomic(tmpName, filename))
      Print("WARNING: Failed atomic replace for training file ", filename);
}
//+------------------------------------------------------------------+
void LoadTrainingData()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_training.csv";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_CSV | FILE_ANSI, ',');
   if(handle == INVALID_HANDLE) return;
   if(FileSize(handle) <= 0)
   {
      RegisterDataWarning("Zero-length persistence file: " + filename);
      FileClose(handle);
      return;
   }

   string schemaKey = FileReadString(handle);
   int schemaVer = (int)FileReadNumber(handle);
   string rowsKey = FileReadString(handle);
   int rowCount = (int)FileReadNumber(handle);
   if(schemaKey != "Schema" || rowsKey != "Rows")
   {
      Print("WARNING: Training CSV header corrupt. Skipping load.");
      FileClose(handle);
      return;
   }

   for(int h = 0; h < 17; h++)
      FileReadString(handle);

   g_trainingDataCount = 0;
   int badRows = 0;
   while(!FileIsEnding(handle) && g_trainingDataCount < INPUT_MAX_TRAINING_DATA)
   {
      TrainingData row;
      row.ticket = (ulong)FileReadNumber(handle);
      row.entryTime = (datetime)FileReadNumber(handle);
      row.closeTime = (datetime)FileReadNumber(handle);
      row.signalCombination = FileReadString(handle);
      row.profitLoss = FileReadNumber(handle);
      row.isWin = ((int)FileReadNumber(handle) == 1);
      row.confidenceAtEntry = FileReadNumber(handle);
      row.threatAtEntry = FileReadNumber(handle);
      row.mtfScore = (int)FileReadNumber(handle);
      row.volatilityRatio = FileReadNumber(handle);
      row.entrySession = (int)FileReadNumber(handle);
      row.closeSession = (int)FileReadNumber(handle);
      row.entryDayOfWeek = (int)FileReadNumber(handle);
      row.closeDayOfWeek = (int)FileReadNumber(handle);
      row.entryRegime = (ENUM_MARKET_REGIME)(int)FileReadNumber(handle);
      row.closeRegime = (ENUM_MARKET_REGIME)(int)FileReadNumber(handle);
      row.fingerprintId = FileReadString(handle);

      if(schemaVer < 3)
      {
         if(row.entryTime > 0)
         {
            row.entrySession = GetSessionFromTime(row.entryTime);
            MqlDateTime edt;
            TimeToStruct(row.entryTime, edt);
            row.entryDayOfWeek = edt.day_of_week;
         }
         if(row.entryRegime == REGIME_UNKNOWN)
            row.entryRegime = row.closeRegime;
      }

      bool valid = (row.ticket > 0 && MathIsValidNumber(row.profitLoss) &&
                    MathIsValidNumber(row.confidenceAtEntry) && row.confidenceAtEntry >= 0.0 && row.confidenceAtEntry <= 100.0 &&
                    MathIsValidNumber(row.threatAtEntry) && row.threatAtEntry >= 0.0 && row.threatAtEntry <= 100.0 &&
                    MathIsValidNumber(row.volatilityRatio) && row.volatilityRatio > 0.0 && row.volatilityRatio < 20.0 &&
                    row.entrySession >= -1 && row.entrySession <= 2 && row.closeSession >= -1 && row.closeSession <= 2 && row.entryDayOfWeek >= 0 && row.entryDayOfWeek <= 6 && row.closeDayOfWeek >= 0 && row.closeDayOfWeek <= 6);
      if(!valid)
      {
         badRows++;
         continue;
      }

      g_trainingData[g_trainingDataCount++] = row;
   }

   FileClose(handle);
   if(schemaVer != TRAINING_SCHEMA_VERSION)
      Print("WARNING: Training schema mismatch file=", schemaVer, " expected=", TRAINING_SCHEMA_VERSION);
   if(schemaVer < TRAINING_SCHEMA_VERSION)
      Print("Training migration applied from schema ", schemaVer, " -> ", TRAINING_SCHEMA_VERSION);
   Print("Training data loaded: ", g_trainingDataCount, " records | badRows=", badRows, " | declaredRows=", rowCount);
}
//+------------------------------------------------------------------+
void SaveAdaptiveParams()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_adaptive.bin";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   FileWriteInteger(handle, ADAPTIVE_SCHEMA_VERSION);
   FileWriteDouble(handle, g_adaptive.lotMultiplier);
   FileWriteDouble(handle, g_adaptive.slAdjustPoints);
   FileWriteDouble(handle, g_adaptive.tpAdjustPoints);
   FileWriteDouble(handle, g_adaptive.trailAdjustPoints);
   FileWriteDouble(handle, g_adaptive.threatMultiplier);
   FileWriteDouble(handle, g_adaptive.confMultiplierCap);
   FileWriteDouble(handle, g_adaptive.minConfThreshold);
   FileWriteInteger(handle, g_adaptive.maxPositions);
   FileWriteInteger(handle, g_totalTrades);

   FileClose(handle);
   if(!ReplaceFileAtomic(tmpName, filename))
      Print("WARNING: Failed atomic replace for adaptive file ", filename);
}
//+------------------------------------------------------------------+
void LoadAdaptiveParams()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_adaptive.bin";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   int version = FileReadInteger(handle);
   if(version <= 0 || version > ADAPTIVE_SCHEMA_VERSION)
   {
      Print("WARNING: Adaptive params schema unsupported: ", version, " - using defaults");
      FileClose(handle);
      ResetAdaptiveParamsToDefaults();
      return;
   }

   g_adaptive.lotMultiplier = FileReadDouble(handle);
   g_adaptive.slAdjustPoints = FileReadDouble(handle);
   g_adaptive.tpAdjustPoints = FileReadDouble(handle);
   g_adaptive.trailAdjustPoints = FileReadDouble(handle);
   g_adaptive.threatMultiplier = FileReadDouble(handle);
   g_adaptive.confMultiplierCap = FileReadDouble(handle);
   g_adaptive.minConfThreshold = FileReadDouble(handle);
   g_adaptive.maxPositions = FileReadInteger(handle);
   g_totalTrades = FileReadInteger(handle);

   FileClose(handle);

   // V7.2 FIX (BUG 1, 2): Validate ALL loaded adaptive params with sane defaults
   // lotMultiplier: 0.5?2.0 (prevent zero or negative)
   if(g_adaptive.lotMultiplier <= 0 || g_adaptive.lotMultiplier > 2.0 || !MathIsValidNumber(g_adaptive.lotMultiplier))
   {
      Print("WARNING: Invalid lotMultiplier=", g_adaptive.lotMultiplier, " resetting to 1.0");
      g_adaptive.lotMultiplier = 1.0;
   }
   g_adaptive.lotMultiplier = MathMax(0.5, MathMin(g_adaptive.lotMultiplier, 2.0));

   // threatMultiplier: 0.5?2.0 (excessive values block all trades)
   if(g_adaptive.threatMultiplier <= 0 || g_adaptive.threatMultiplier > 2.0 || !MathIsValidNumber(g_adaptive.threatMultiplier))
   {
      Print("WARNING: Invalid threatMultiplier=", g_adaptive.threatMultiplier, " resetting to 1.0");
      g_adaptive.threatMultiplier = 1.0;
   }
   g_adaptive.threatMultiplier = MathMax(0.5, MathMin(g_adaptive.threatMultiplier, 2.0));

   // minConfThreshold: 20?80 (100?% would block everything)
   if(g_adaptive.minConfThreshold < 0 || g_adaptive.minConfThreshold > 100 || !MathIsValidNumber(g_adaptive.minConfThreshold))
   {
      Print("WARNING: Invalid minConfThreshold=", g_adaptive.minConfThreshold,
            " resetting to ", INPUT_MIN_CONFIDENCE);
      g_adaptive.minConfThreshold = INPUT_MIN_CONFIDENCE;
   }
   g_adaptive.minConfThreshold = MathMax(20.0, MathMin(g_adaptive.minConfThreshold, 80.0));

   // maxPositions: at least 2 (0?? gate always blocks)
   if(g_adaptive.maxPositions <= 0 || g_adaptive.maxPositions > INPUT_MAX_CONCURRENT_TRADES + 5)
   {
      Print("WARNING: Invalid maxPositions=", g_adaptive.maxPositions,
            " resetting to ", INPUT_MAX_CONCURRENT_TRADES);
      g_adaptive.maxPositions = INPUT_MAX_CONCURRENT_TRADES;
   }
   g_adaptive.maxPositions = MathMax(g_adaptive.maxPositions, 2);

   // confMultiplierCap: 1.0?2.0
   if(g_adaptive.confMultiplierCap <= 0 || g_adaptive.confMultiplierCap > 3.0 || !MathIsValidNumber(g_adaptive.confMultiplierCap))
   {
      Print("WARNING: Invalid confMultiplierCap=", g_adaptive.confMultiplierCap,
            " resetting to 1.5");
      g_adaptive.confMultiplierCap = 1.5;
   }
   g_adaptive.confMultiplierCap = MathMax(1.0, MathMin(g_adaptive.confMultiplierCap, 2.0));

   Print("Adaptive params loaded and VALIDATED. Total trades: ", g_totalTrades,
         " | lotMult=", g_adaptive.lotMultiplier,
         " | threatMult=", g_adaptive.threatMultiplier,
         " | minConf=", g_adaptive.minConfThreshold,
         " | maxPos=", g_adaptive.maxPositions);
}
void SaveRuntimeState()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_runtime.bin";
   string tmpName = filename + ".tmp";
   int handle = FileOpen(tmpName, FILE_WRITE | FILE_BIN);
   if(handle == INVALID_HANDLE)
   {
      Print("WARNING: SaveRuntimeState failed to open ", tmpName);
      return;
   }


   if(INPUT_RL_PENDING_HARD_CAP < 0)
      Print("WARNING: INPUT_RL_PENDING_HARD_CAP is negative (", INPUT_RL_PENDING_HARD_CAP, ") - clamped to 0 for serialization.");

   FileWriteInteger(handle, RUNTIME_SCHEMA_VERSION);
   FileWriteLong(handle, (long)g_lastProcessedDealTicket);
   FileWriteLong(handle, (long)g_lastProcessedDealTime);
   FileWriteLong(handle, (long)g_lastProcessedEntryDealTicket);
   FileWriteLong(handle, (long)g_lastProcessedEntryDealTime);
   FileWriteInteger(handle, g_rlMatchedUpdates);
   FileWriteInteger(handle, g_rlUnmatchedCloses);
   FileWriteInteger(handle, g_closedDealsProcessedTotal);

   int pendingToWrite = MathMin(g_pendingRLCount, MathMax(0, INPUT_RL_PENDING_HARD_CAP));
   FileWriteInteger(handle, pendingToWrite);

   uint checksum = FNV1aStart();
   checksum = FNV1aUpdateInt(checksum, RUNTIME_HASH_SENTINEL);
   checksum = FNV1aUpdateInt(checksum, RUNTIME_SCHEMA_VERSION);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedDealTicket);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedDealTime);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedEntryDealTicket);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedEntryDealTime);
   checksum = FNV1aUpdateInt(checksum, g_rlMatchedUpdates);
   checksum = FNV1aUpdateInt(checksum, g_rlUnmatchedCloses);
   checksum = FNV1aUpdateInt(checksum, g_closedDealsProcessedTotal);
   checksum = FNV1aUpdateInt(checksum, pendingToWrite);

   for(int i = 0; i < pendingToWrite; i++)
   {
      FileWriteInteger(handle, g_pendingRL[i].state);
      FileWriteInteger(handle, (int)g_pendingRL[i].action);
      FileWriteLong(handle, (long)g_pendingRL[i].timestamp);
      FileWriteLong(handle, (long)g_pendingRL[i].orderTicket);
      FileWriteLong(handle, (long)g_pendingRL[i].positionTicket);
      FileWriteDouble(handle, g_pendingRL[i].entryPrice);
      FileWriteDouble(handle, g_pendingRL[i].slDistance);
      FileWriteDouble(handle, g_pendingRL[i].lot);
      FileWriteDouble(handle, g_pendingRL[i].tickValue);
      FileWriteDouble(handle, g_pendingRL[i].confidenceSnapshot);
      FileWriteInteger(handle, g_pendingRL[i].mtfScoreSnapshot);
      FileWriteDouble(handle, g_pendingRL[i].comboStrengthSnapshot);

      checksum = FNV1aUpdateInt(checksum, g_pendingRL[i].state);
      checksum = FNV1aUpdateInt(checksum, (int)g_pendingRL[i].action);
      checksum = FNV1aUpdateLong(checksum, (long)g_pendingRL[i].timestamp);
      checksum = FNV1aUpdateLong(checksum, (long)g_pendingRL[i].orderTicket);
      checksum = FNV1aUpdateLong(checksum, (long)g_pendingRL[i].positionTicket);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].entryPrice);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].slDistance);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].lot);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].tickValue);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].confidenceSnapshot);
      checksum = FNV1aUpdateInt(checksum, g_pendingRL[i].mtfScoreSnapshot);
      checksum = FNV1aUpdateDouble(checksum, g_pendingRL[i].comboStrengthSnapshot);
   }

   FileWriteInteger(handle, (int)checksum);
   FileFlush(handle);
   FileClose(handle);

   if(!ReplaceFileAtomic(tmpName, filename))
      Print("ERROR: SaveRuntimeState replace failed, previous runtime file kept intact: ", filename);
}

void ResetRuntimeStateAfterIntegrityFailure()
{
   ResetRuntimeLinkedInMemoryState();
}

void LoadRuntimeState()
{
   string filename = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER) + "_runtime.bin";
   if(!FileIsExist(filename)) return;

   int handle = FileOpen(filename, FILE_READ | FILE_BIN);
   if(handle == INVALID_HANDLE)
   {
      Print("WARNING: LoadRuntimeState failed to open ", filename);
      return;
   }

   int version = FileReadInteger(handle);
   if(version < 1 || version > RUNTIME_SCHEMA_VERSION)
   {
      Print("WARNING: Runtime state version mismatch: ", version);
      FileClose(handle);
      return;
   }

   g_lastProcessedDealTicket = (ulong)FileReadLong(handle);
   g_lastProcessedDealTime = (version >= 3) ? (datetime)FileReadLong(handle) : 0;
   g_lastProcessedEntryDealTicket = (version >= 4) ? (ulong)FileReadLong(handle) : 0;
   g_lastProcessedEntryDealTime = (version >= 4) ? (datetime)FileReadLong(handle) : 0;
   g_rlMatchedUpdates = FileReadInteger(handle);
   g_rlUnmatchedCloses = FileReadInteger(handle);
   g_closedDealsProcessedTotal = FileReadInteger(handle);

   int pendingRead = (version >= 2) ? FileReadInteger(handle) : 0;
   if(pendingRead < 0) pendingRead = 0;
   if(INPUT_RL_PENDING_HARD_CAP < 0)
      Print("WARNING: INPUT_RL_PENDING_HARD_CAP is negative (", INPUT_RL_PENDING_HARD_CAP, ") - clamped to 0 during load.");

   uint checksum = FNV1aStart();
   checksum = FNV1aUpdateInt(checksum, RUNTIME_HASH_SENTINEL);
   checksum = FNV1aUpdateInt(checksum, version);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedDealTicket);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedDealTime);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedEntryDealTicket);
   checksum = FNV1aUpdateLong(checksum, (long)g_lastProcessedEntryDealTime);
   checksum = FNV1aUpdateInt(checksum, g_rlMatchedUpdates);
   checksum = FNV1aUpdateInt(checksum, g_rlUnmatchedCloses);
   checksum = FNV1aUpdateInt(checksum, g_closedDealsProcessedTotal);
   checksum = FNV1aUpdateInt(checksum, pendingRead);

   g_pendingRLCount = 0;
   int pendingCap = MathMax(0, INPUT_RL_PENDING_HARD_CAP);
   if(ArraySize(g_pendingRL) < pendingCap) ArrayResize(g_pendingRL, pendingCap);

   for(int i = 0; i < pendingRead; i++)
   {
      RLStateAction rec;
      rec.state = FileReadInteger(handle);
      rec.action = (ENUM_RL_ACTION)FileReadInteger(handle);
      rec.timestamp = (datetime)FileReadLong(handle);
      rec.orderTicket = (version >= 4) ? (ulong)FileReadLong(handle) : 0;
      rec.positionTicket = (ulong)FileReadLong(handle);
      rec.entryPrice = FileReadDouble(handle);
      rec.slDistance = FileReadDouble(handle);
      rec.lot = FileReadDouble(handle);
      rec.tickValue = FileReadDouble(handle);
      rec.confidenceSnapshot = (version >= 6) ? FileReadDouble(handle) : 50.0;
      rec.mtfScoreSnapshot = (version >= 6) ? FileReadInteger(handle) : 0;
      rec.comboStrengthSnapshot = (version >= 6) ? FileReadDouble(handle) : 50.0;

      checksum = FNV1aUpdateInt(checksum, rec.state);
      checksum = FNV1aUpdateInt(checksum, (int)rec.action);
      checksum = FNV1aUpdateLong(checksum, (long)rec.timestamp);
      checksum = FNV1aUpdateLong(checksum, (long)rec.orderTicket);
      checksum = FNV1aUpdateLong(checksum, (long)rec.positionTicket);
      checksum = FNV1aUpdateDouble(checksum, rec.entryPrice);
      checksum = FNV1aUpdateDouble(checksum, rec.slDistance);
      checksum = FNV1aUpdateDouble(checksum, rec.lot);
      checksum = FNV1aUpdateDouble(checksum, rec.tickValue);
      checksum = FNV1aUpdateDouble(checksum, rec.confidenceSnapshot);
      checksum = FNV1aUpdateInt(checksum, rec.mtfScoreSnapshot);
      checksum = FNV1aUpdateDouble(checksum, rec.comboStrengthSnapshot);

      bool validRec = (rec.state >= 0 && rec.state < Q_TABLE_STATES && rec.action >= 0 && rec.action < Q_TABLE_ACTIONS &&
                       MathIsValidNumber(rec.entryPrice) && MathIsValidNumber(rec.slDistance) &&
                       MathIsValidNumber(rec.lot) && MathIsValidNumber(rec.tickValue));
      if(validRec && rec.orderTicket > 0 && rec.positionTicket == 0 && !OrderSelect(rec.orderTicket))
         validRec = false;

      if(validRec && g_pendingRLCount < pendingCap)
         g_pendingRL[g_pendingRLCount++] = rec;
      else if(!validRec)
         RegisterDataWarning("Runtime pending RL record dropped during load");
   }

   int fileChecksum = (version >= 5) ? FileReadInteger(handle) : (int)checksum;
   FileClose(handle);

   bool checksumOk = ((int)checksum == fileChecksum);
   if(!checksumOk)
   {
      Print("WARNING: Runtime state checksum mismatch expected=", fileChecksum, " computed=", (int)checksum);
      RegisterDataWarning("Runtime checksum mismatch");
      if(INPUT_STRICT_STATE_LOAD)
      {
         Print("STRICT LOAD: Discarding runtime RL pending state and resetting watermarks.");
         ResetRuntimeStateAfterIntegrityFailure();
      }
   }
}

void ResetRuntimeLinkedInMemoryState()
{
   g_pendingRLCount = 0;
   ArrayResize(g_pendingRL, MathMax(0, INPUT_RL_PENDING_HARD_CAP));
   ArrayResize(g_markovQueue, 0);
   g_markovQueueCount = 0;
   g_markovQueueHead = 0;
   g_lastProcessedDealTicket = 0;
   g_lastProcessedDealTime = 0;
   g_lastProcessedEntryDealTicket = 0;
   g_lastProcessedEntryDealTime = 0;
   g_rlMatchedUpdates = 0;
   g_rlUnmatchedCloses = 0;
   g_closedDealsProcessedTotal = 0;
}

bool ResetAllPersistedStateFiles()
{
   string base = _Symbol + "_" + IntegerToString(INPUT_MAGIC_NUMBER);
   string files[] = {
      base + "_training.csv",
      base + "_fp.csv",
      base + "_combo_stats.csv",
      base + "_qtable.bin",
      base + "_markov.bin",
      base + "_adaptive.bin",
      base + "_runtime.bin",
      base + "_closed_deals.csv"
   };

   bool success = true;
   int deleted = 0;
   for(int i = 0; i < ArraySize(files); i++)
   {
      if(FileIsExist(files[i]))
      {
         bool ok = FileDelete(files[i]);
         Print("RESET PERSISTENCE: ", (ok ? "deleted " : "FAILED "), files[i]);
         if(ok) deleted++;
         if(!ok) success = false;
      }
      string tmp = files[i] + ".tmp";
      if(FileIsExist(tmp))
      {
         bool okTmp = FileDelete(tmp);
         Print("RESET PERSISTENCE: ", (okTmp ? "deleted " : "FAILED "), tmp);
         if(!okTmp) success = false;
      }
      string bak = files[i] + ".bak";
      if(FileIsExist(bak))
      {
         bool okBak = FileDelete(bak);
         Print("RESET PERSISTENCE: ", (okBak ? "deleted " : "FAILED "), bak);
         if(!okBak) success = false;
      }
   }

   g_fingerprintCount = 0;
   g_trainingDataCount = 0;
   g_combinationStatsCount = 0;
   g_comboObservedCount = 0;
   g_markovTradesRecorded = 0;
   g_lastMarkovState = MARKOV_EVEN;
   ResetRuntimeLinkedInMemoryState();

   Print("RESET PERSISTENCE: completed deletedFiles=", deleted, " success=", (success ? "true" : "false"));
   return success;
}

//+------------------------------------------------------------------+
//| SECTION 31: CHART PANEL                                          |
//+------------------------------------------------------------------+
void DrawStatsPanel()
{
   static datetime lastPanelUpdate = 0;
   if(TimeCurrent() - lastPanelUpdate < 5) return; // Throttle to 5?seconds
   lastPanelUpdate = TimeCurrent();

   int x = 10, y = 30;
   color bgColor = clrDarkSlateGray;
   color txtColor = clrWhite;

   string prefix = "V7_Panel_";

   // Background rectangle (create once, then update)
   if(ObjectFind(0, prefix + "bg") < 0)
      ObjectCreate(0, prefix + "bg", OBJ_RECTANGLE_LABEL, 0, 0, 0);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_XSIZE, 360);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_YSIZE, 430);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_BGCOLOR, bgColor);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_BORDER_TYPE, BORDER_FLAT);
   ObjectSetInteger(0, prefix + "bg", OBJPROP_CORNER, CORNER_LEFT_UPPER);

   y += 10;

   // Title
   CreateLabel(prefix + "title", "EA " + EA_VERSION_LABEL + " HumanBrain", x + 10, y, clrGold, 10);
   y += 20;

   // State
   string stateStr = EnumToString(g_eaState);
   color stateColor = clrLime;
   if(g_eaState == STATE_EXTREME_RISK) stateColor = clrRed;
   else if(g_eaState == STATE_RECOVERY_ACTIVE) stateColor = clrOrange;
   else if(g_eaState == STATE_DRAWDOWN_PROTECT) stateColor = clrYellow;

   CreateLabel(prefix + "state", "State: " + stateStr, x + 10, y, stateColor, 9);
   y += 18;

   // Threat
   double threat = CalculateMarketThreat();
   ENUM_THREAT_ZONE zone = GetThreatZone(threat);
   color thColor = clrLime;
   if(zone == THREAT_EXTREME) thColor = clrRed;
   else if(zone == THREAT_RED)    thColor = clrOrangeRed;
   else if(zone == THREAT_ORANGE)thColor = clrOrange;
   else if(zone == THREAT_YELLOW)thColor = clrYellow;

   CreateLabel(prefix + "threat", "Threat: " + DoubleToString(threat,1) + " (" +
               EnumToString(zone) + ")", x + 10, y, thColor, 9);
   y += 18;

   // Regime
   CreateLabel(prefix + "regime", "Regime: " + EnumToString(g_currentRegime), x + 10, y, txtColor, 9);
   y += 18;

   // Position counts (broker counts)
   int mainPos = CountMainPositionsFromBroker();
   int totalPos = CountAllOurPositions();
   double balance = AccountInfoDouble(ACCOUNT_BALANCE);
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   double openPL = GetOpenProfitLoss();
   color openPLColor = openPL >= 0 ? clrLime : clrRed;
   CreateLabel(prefix + "account", "Bal: " + DoubleToString(balance, 2) +
               " | Eq: " + DoubleToString(equity, 2) +
               " | Open P/L: " + DoubleToString(openPL, 2),
               x + 10, y, openPLColor, 9);
   y += 18;

   CreateLabel(prefix + "positions", "Positions: " + IntegerToString(mainPos) + " main / " +
               IntegerToString(totalPos) + " total (max " + IntegerToString(g_adaptive.maxPositions) + ")",
               x + 10, y, txtColor, 9);
   y += 18;

   CreateLabel(prefix + "master_toggles", "Toggles P/C/M: " +
               (INPUT_TOGGLE_PLACE_ORDERS ? "ON" : "OFF") + "/" +
               (INPUT_TOGGLE_CLOSE_ORDERS ? "ON" : "OFF") + "/" +
               ((INPUT_TOGGLE_MODIFY_STOPS || INPUT_TOGGLE_MODIFY_TPS) ? "ON" : "OFF"),
               x + 10, y, txtColor, 9);
   y += 18;

   // Daily stats
   CreateLabel(prefix + "daily", "Today: Filled " + IntegerToString(g_daily.tradesPlaced) +
               " | Pending Placed " + IntegerToString(g_daily.pendingOrdersPlaced) +
               " | W:" + IntegerToString(g_daily.winsToday) + " L:" + IntegerToString(g_daily.lossesToday),
               x + 10, y, txtColor, 9);
   y += 18;

   CreateLabel(prefix + "activity", "Open Positions: " + IntegerToString(totalPos) +
               " | Closed Deals Today: " + IntegerToString(g_daily.closedDealsToday) +
               " | Closed Processed: " + IntegerToString(g_closedDealsProcessedTotal),
               x + 10, y, txtColor, 9);
   y += 18;

   // P&L today
   double netToday = g_daily.profitToday - g_daily.lossToday;
   color plColor = netToday >= 0 ? clrLime : clrRed;
   CreateLabel(prefix + "pnl", "P&L Today: " + DoubleToString(netToday, 2), x + 10, y, plColor, 9);
   y += 18;

   double maxDayLoss = g_daily.dayStartBalance * (INPUT_DAILY_LOSS_LIMIT_PERCENT / 100.0);
   CreateLabel(prefix + "dailycap", "Daily loss cap uses dayStartBalance=" + DoubleToString(g_daily.dayStartBalance, 2) +
               " (limit " + DoubleToString(maxDayLoss, 2) + ")", x + 10, y, txtColor, 9);
   y += 18;

   // Streak
   CreateLabel(prefix + "streak", "Streak: W" + IntegerToString(g_consecutiveWins) +
               " / L" + IntegerToString(g_consecutiveLosses) +
               " | Trigger=" + IntegerToString(INPUT_STREAK_TRIGGER_WINS) +
               " | Mult=" + DoubleToString(INPUT_STREAK_LOT_MULTIPLIER, 2) +
               " | BoostLeft=" + IntegerToString(g_streakMultiplierOrdersRemaining),
               x + 10, y, txtColor, 9);
   y += 18;

   // Drawdown
   double dd = CalculateDrawdownPercent();
   color ddColor = dd < 1 ? clrLime : (dd < 2 ? clrYellow : clrRed);
   CreateLabel(prefix + "dd", "Drawdown: " + DoubleToString(dd,2) + "%", x + 10, y, ddColor, 9);
   y += 18;

   // Adaptive params
   CreateLabel(prefix + "adapt", "Lot Mult: " + DoubleToString(g_adaptive.lotMultiplier,2) +
               " | Min Conf: " + DoubleToString(g_adaptive.minConfThreshold,1) + "%",
               x + 10, y, txtColor, 9);
   y += 18;

      // Total trades
   CreateLabel(prefix + "total", "Total Trades: " + IntegerToString(g_totalTrades), x + 10, y, txtColor, 9);
   y += 18;

   // Gate diagnostics
   CreateLabel(prefix + "diagSession", "Rejects Session/Cooldown: " +
               IntegerToString(g_gateDiagnostics.sessionRejects) + " / " +
               IntegerToString(g_gateDiagnostics.cooldownRejects), x + 10, y, txtColor, 9);
   y += 18;
   CreateLabel(prefix + "diagSignals", "Rejects Signals/MTF: " +
               IntegerToString(g_gateDiagnostics.signalsRejects) + " / " +
               IntegerToString(g_gateDiagnostics.mtfRejects), x + 10, y, txtColor, 9);
   y += 18;
     CreateLabel(prefix + "diagData", "DataRejects MTF/ADX: " +
               IntegerToString(g_gateDiagnostics.mtfDataReadRejects) + " / " +
               IntegerToString(g_gateDiagnostics.adxDataReadRejects) +
               " | MTFReadFailTick=" + (g_mtfReadFailureThisTick ? "Y" : "N"), x + 10, y, txtColor, 9);
   y += 18;
   CreateLabel(prefix + "diagThreat", "Rejects Threat/Confidence: " +
               IntegerToString(g_gateDiagnostics.threatRejects) + " / " +
               IntegerToString(g_gateDiagnostics.confidenceRejects), x + 10, y, txtColor, 9);
   y += 18;
   CreateLabel(prefix + "diagMaxPos", "Rejects Max Positions: " +
               IntegerToString(g_gateDiagnostics.maxPositionsRejects), x + 10, y, txtColor, 9);
   y += 18;
   // RL info
   if(INPUT_ENABLE_RL)
   {
      CreateLabel(prefix + "rl", "RL Trades: " + IntegerToString(g_rlTradesCompleted) +
                  " | Pending:" + IntegerToString(g_pendingRLCount), x + 10, y, clrCyan, 9);
      y += 18;
      CreateLabel(prefix + "rldiag", "RL Matched:" + IntegerToString(g_rlMatchedUpdates) +
                  " Unmatched:" + IntegerToString(g_rlUnmatchedCloses), x + 10, y, clrCyan, 9);
      y += 18;
   }

   // ML info
   if(INPUT_ENABLE_ML)
   {
      CreateLabel(prefix + "ml", "Training Data: " + IntegerToString(g_trainingDataCount), x + 10, y, clrCyan, 9);
      y += 18;
   }
   CreateLabel(prefix + "comboCov", "Combo coverage: " + IntegerToString(g_comboObservedCount) + "/" + IntegerToString(MathMax(g_combinationStatsCount,1)), x + 10, y, clrCyan, 9);
   y += 18;

   // Settings summary
   CreateLabel(prefix + "settings", "MinSig:" + IntegerToString(INPUT_MIN_SIGNALS) +
               " MTF:" + IntegerToString(INPUT_MIN_MTF_SCORE) +
               " ADX:" + (INPUT_USE_ADX_FILTER ? "ON" : "OFF"),
               x + 10, y, clrGray, 8);
   y += 16;

   // Version line
   CreateLabel(prefix + "version", EA_VERSION_LABEL + " runtime", x + 10, y, clrGray, 8);

   ChartRedraw(0);
}
//+------------------------------------------------------------------+
void CreateLabel(string name, string text, int x, int y, color clr, int fontSize)
{
   if(ObjectFind(0, name) < 0)
      ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, fontSize);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
}
//+------------------------------------------------------------------+
//+------------------------------------------------------------------+
//| V7.6 FIX: Advanced Filling Mode Detection (Critical Bug 2)      |
//+------------------------------------------------------------------+
ENUM_ORDER_TYPE_FILLING GetFillingMode()
{
   long filling = SymbolInfoInteger(_Symbol, SYMBOL_FILLING_MODE);
   if((filling & SYMBOL_FILLING_FOK) != 0) return ORDER_FILLING_FOK;
   if((filling & SYMBOL_FILLING_IOC) != 0) return ORDER_FILLING_IOC;
   #ifdef SYMBOL_FILLING_RETURN
   if((filling & SYMBOL_FILLING_RETURN) != 0) return ORDER_FILLING_RETURN;
#endif

    // Broker did not advertise known modes; keep IOC fallback for safety.
   return ORDER_FILLING_IOC;
}

//+------------------------------------------------------------------+
//| V7.6 FIX: Position Age Timeout Implementation (Critical Bug 4)   |
//+------------------------------------------------------------------+
void CheckPositionAgeTimeout()
{
   if(!IsCloseEnabled() || !INPUT_CLOSE_AGE_TIMEOUT_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effClosePositionAgeTimeout)
   {
      if(!loggedDisabled)
      {
         Print("POSITION AGE TIMEOUT close disabled (INPUT_ENABLE_CLOSE_POSITION_AGE_TIMEOUT=OFF)");
         loggedDisabled = true;
      }
      return;
   }

   if(INPUT_POSITION_AGE_HOURS <= 0) return;

   datetime now = TimeCurrent();
   if(g_lastPositionAgeCheck > 0 && (now - g_lastPositionAgeCheck) < INPUT_POSITION_AGE_CHECK_SECONDS)
      return;
   g_lastPositionAgeCheck = now;

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
       if(ticket == 0) continue;
      if(!PositionSelectByTicket(ticket) || !IsOurPosition(ticket)) continue;

      string comment = PositionGetString(POSITION_COMMENT);
      long magic = PositionGetInteger(POSITION_MAGIC);
      bool isAux = IsAuxSubtypeByMagic(magic) ||
                   (StringFind(comment, COMMENT_RECOVERY_PREFIX) >= 0 ||
                    StringFind(comment, COMMENT_AVG_PREFIX)      >= 0 ||
                    StringFind(comment, COMMENT_HEDGE_PREFIX)    >= 0);
      if(isAux && !INPUT_AGE_TIMEOUT_INCLUDE_AUX)
      {
         if(INPUT_ENABLE_LOGGING)
            Print("AGE TIMEOUT SKIP: aux/recovery position skipped | ticket=", ticket,
                  " | comment=", comment,
                  " | includeAux=", (INPUT_AGE_TIMEOUT_INCLUDE_AUX ? "true" : "false"));
         continue;
      }
      datetime openTime = (datetime)PositionGetInteger(POSITION_TIME);
      if(now - openTime > INPUT_POSITION_AGE_HOURS * 3600)
      {
         Print("Closing stale position (Age Timeout): ", ticket,
               " | ageHours=", DoubleToString((now - openTime) / 3600.0, 2));
         g_trade.PositionClose(ticket);
      }
   }
}

//+------------------------------------------------------------------+
//| V7.6 FIX: Trailing Take Profit (Missing Feature 11)              |
//+------------------------------------------------------------------+
void ManageTrailingTP()
{
   if(!IsTpModifyEnabled() || !INPUT_MODIFY_TRAILING_TP_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effModifyTrailingTP)
   {
      if(!loggedDisabled)
      {
         Print("TRAILING TP modify disabled (INPUT_ENABLE_MODIFY_TRAILING_TP=OFF or legacy OFF)");
         loggedDisabled = true;
      }
      return;
   }

   datetime now = TimeCurrent();
   if(g_lastTrailingTPCheck > 0 && (now - g_lastTrailingTPCheck) < 5)
      return;
   g_lastTrailingTPCheck = now;

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!IsOurPosition(ticket)) continue;
      if(!PositionSelectByTicket(ticket)) continue;

      // High-spread policy: close winners immediately, do not touch losing-position stops/TP.
      if(ShouldSkipStopAdjustmentsForTicket(ticket))
         continue;

      if(!CanModifyPosition(ticket))
         continue;

      if(!CanAttemptTPModify(ticket))
         continue;

      ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
      double tp = PositionGetDouble(POSITION_TP);
      double sl = PositionGetDouble(POSITION_SL);
      double current = PositionGetDouble(POSITION_PRICE_CURRENT);
      double entry = PositionGetDouble(POSITION_PRICE_OPEN);
      int dir = (posType == POSITION_TYPE_BUY) ? 1 : -1;

     

      double profitPoints = dir * (current - entry) / _Point;
      if(profitPoints < INPUT_TRAIL_ACTIVATION_POINTS)
         continue;

      double newTP = current + dir * INPUT_TRAIL_STEP_POINTS * _Point;

      // Normalize and enforce minimum distance by stop-level/freeze-level constraints
      newTP = NormalizeDouble(newTP, g_digits);
      double minDistPoints = (double)MathMax(g_stopLevel, g_freezeLevel);
      double minDistPrice = minDistPoints * g_point;
      if(minDistPrice > 0)
      {
         if(dir == 1 && (newTP - current) < minDistPrice)
            newTP = NormalizeDouble(current + minDistPrice, g_digits);
         else if(dir == -1 && (current - newTP) < minDistPrice)
            newTP = NormalizeDouble(current - minDistPrice, g_digits);
      }

       // BUY: TP should move up; SELL: TP should move down (further away in profit direction).
      // If TP is not set yet (trailing-only mode), first TP assignment is always allowed.
      bool hasExistingTP = (tp > 0.0);
      bool shouldMove = hasExistingTP ? ((dir == 1) ? (newTP > tp) : (newTP < tp)) : true;
      if(!shouldMove)
         continue;

      if(g_trade.PositionModify(ticket, sl, newTP))
      {
         ResetTPFailureTracker(ticket);
      }
      else
      {
         RegisterTPModifyFailure(ticket);
         LogWithRestartGuard("TRAIL TP MODIFY FAILED: ticket=" + IntegerToString((int)ticket) +
                            " | retcode=" + IntegerToString((int)g_trade.ResultRetcode()) +
                            " | comment=" + g_trade.ResultComment());
      }
   }
}

//+------------------------------------------------------------------+
//| V7.6 FIX: Multi-Level Partial Close (Missing Feature 12)        |
//+------------------------------------------------------------------+
void HandleMultiLevelPartial(ulong ticket)
{
   if(!IsCloseEnabled() || !INPUT_CLOSE_MULTI_LEVEL_PARTIAL_ON)
      return;
   static bool loggedDisabled = false;
   if(!g_effCloseMultiLevelPartial)
   {
      if(!loggedDisabled)
      {
         Print("MULTI LEVEL PARTIAL close disabled (INPUT_ENABLE_CLOSE_MULTI_LEVEL_PARTIAL=OFF)");
         loggedDisabled = true;
      }
      return;
   }

    if(ticket == 0) return;
   if(!PositionSelectByTicket(ticket)) return;
   if(!IsOurPosition(ticket)) return;

   int idx = -1;
   for(int i = 0; i < g_positionCount; i++)
   {
      if(g_positions[i].isActive && g_positions[i].ticket == ticket)
      {
         idx = i;
         break;
      }
   }
   if(idx < 0) return;

   // Avoid conflicts with other partial-close systems
   if(g_positions[idx].lotReduced || g_positions[idx].partialClosed)
      return;
   double vol = PositionGetDouble(POSITION_VOLUME);
   double open = PositionGetDouble(POSITION_PRICE_OPEN);
   double tp = PositionGetDouble(POSITION_TP);
   double current = PositionGetDouble(POSITION_PRICE_CURRENT);
   ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);

   double totalDist = MathAbs(tp - open);
   if(totalDist <= 0) 
   {
      if(INPUT_ENABLE_LOGGING)
         Print("MULTI PARTIAL: Invalid TP distance | ticket=", ticket, " | totalDist=", totalDist);
      return;
   }
   
   // V7.31 FIX #2: Directional progress calculation
   double progress = 0.0;
   if(posType == POSITION_TYPE_BUY)
   {
      // BUY: progress when current > open, moving toward TP
      if(tp > open) // valid upward TP
         progress = (current - open) / (tp - open);
   }
   else // SELL
   {
      // SELL: progress when current < open, moving toward TP
      if(tp < open) // valid downward TP
         progress = (open - current) / (open - tp);
   }
   
   // Guard: require progress > 0 before considering thresholds
   if(progress <= 0)
   {
      return; // No progress or moving away from TP
   }

   bool shouldClose = false;
   if(progress >= 0.60 && !g_positions[idx].multiPartialLevel2Done)
      shouldClose = true;
   else if(progress >= 0.30 && !g_positions[idx].multiPartialLevel1Done)
      shouldClose = true;

   if(!shouldClose) return;

   double lotsToClose = vol * 0.25;
  lotsToClose = MathFloor(lotsToClose / GetEffectiveLotStep()) * GetEffectiveLotStep();
   lotsToClose = NormalizeDouble(lotsToClose, (int)g_lotDigits);

   if(lotsToClose < g_minLot || lotsToClose >= vol)
      return;

   if(g_trade.PositionClosePartial(ticket, lotsToClose))
   {
      double newLots = PositionGetDouble(POSITION_VOLUME);
      if(newLots <= 0 || newLots >= vol)
         newLots = vol - lotsToClose;
      g_positions[idx].currentLots = MathMax(0.0, newLots);

      if(progress >= 0.60)
      {
         g_positions[idx].multiPartialLevel2Done = true;
         g_positions[idx].multiPartialLevel1Done = true;
      }
      else if(progress >= 0.30)
      {
         g_positions[idx].multiPartialLevel1Done = true;
      }

      if(INPUT_ENABLE_LOGGING)
         Print("MULTI PARTIAL CLOSE: Ticket ", ticket,
               " | Progress=", DoubleToString(progress, 2),
               " | Closed=", lotsToClose,
               " | Remaining=", g_positions[idx].currentLots,
               " | L1Done=", g_positions[idx].multiPartialLevel1Done,
               " | L2Done=", g_positions[idx].multiPartialLevel2Done);
   }
}
