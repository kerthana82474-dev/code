//+------------------------------------------------------------------+
//|                        XAUUSD_Adaptive_MDAE_EA.mq5               |
//|       Production-Grade Adaptive MDAE Expert Advisor for MT5      |
//|           Version 1.0 — Full Implementation, No Placeholders     |
//+------------------------------------------------------------------+
#property copyright "XAUUSD Adaptive MDAE"
#property link      ""
#property version   "1.00"
#property strict
#include <Trade\Trade.mqh>
//+------------------------------------------------------------------+
//| ENUMERATIONS                                                     |
//+------------------------------------------------------------------+
enum ENUM_REGIME
{
   REGIME_NO_TRADE       = 0,
   REGIME_TREND_EXPANSION= 1,
   REGIME_MEAN_REVERSION = 2
};
enum ENUM_CLOSE_METHOD
{
   CM_FIXED   = 0,
   CM_ATR     = 1,
   CM_SWING   = 2,
   CM_FAST    = 3,
   CM_EXIT    = 4,
   CM_HYBRID  = 5
};
enum ENUM_RISK_STATE
{
   RISK_ALLOW  = 0,
   RISK_REDUCE = 1,
   RISK_BLOCK  = 2
};
//+------------------------------------------------------------------+
//| INPUTS — MASTER TOGGLES                                          |
//+------------------------------------------------------------------+
input group "=== Master Toggles ==="
input bool   EnableEA                     = true;
input bool   EnableEntries                = true;
input bool   EnablePositionManagement     = true;
input bool   EnableLogging                = true;
input bool   EnableDebugPrints            = false;
//+------------------------------------------------------------------+
//| INPUTS — REGIME / ENTRY TOGGLES                                  |
//+------------------------------------------------------------------+
input group "=== Regime & Entry Toggles ==="
input bool   EnableRegimeEngine           = true;
input bool   EnableTrendModule            = true;
input bool   EnableMeanReversionModule    = true;
input bool   EnableNoTradeState           = true;
input bool   EnableMTFContextCheck        = true;
input bool   EnableVWAPCheck              = false;   // fallback if unavailable
input bool   EnableSessionFilter          = true;
input bool   EnableNewsWindowBlock        = false;   // manual schedule
//+------------------------------------------------------------------+
//| INPUTS — RISK / EXECUTION TOGGLES                                |
//+------------------------------------------------------------------+
input group "=== Risk & Execution Toggles ==="
input bool   EnableRiskGovernor           = true;
input bool   EnableDailyDDLock            = true;
input bool   EnableWeeklyDDLock           = true;
input bool   EnableConsecutiveLossLock    = true;
input bool   EnableSpreadEntryFilter      = true;
input bool   EnableSlippageEntryFilter    = true;
input bool   EnableOnePositionOnly        = true;
input bool   EnableMaxTradesPerSession    = true;
//+------------------------------------------------------------------+
//| INPUTS — MDAE FEATURE TOGGLES                                    |
//+------------------------------------------------------------------+
input group "=== MDAE Feature Toggles ==="
input bool   EnableMDAE                   = true;
input bool   EnablePipsPerSecond          = true;
input bool   EnableVolumePulse            = true;
input bool   EnableBodyExpansion          = true;
input bool   EnableTop2BreakoutCheck      = true;
input bool   EnableLowVolumeBox           = true;
input bool   EnableFibGuidance            = false;
input bool   EnableEntropyScoring         = true;
input bool   EnablePolicySwitching        = true;
//+------------------------------------------------------------------+
//| INPUTS — EXIT / ACTION TOGGLES                                   |
//+------------------------------------------------------------------+
input group "=== Exit & Action Toggles ==="
input bool   EnablePartialClose           = true;
input bool   EnableBreakEven              = true;
input bool   EnableATRTrail               = true;
input bool   EnableSwingTrail             = true;
input bool   EnableFastBankMode           = true;
input bool   EnableHybridCloseMode        = true;
input bool   EnableInvalidationExit       = true;
input bool   EnableTimeStop               = true;
//+------------------------------------------------------------------+
//| INPUTS — COMPLIANCE TOGGLES                                      |
//+------------------------------------------------------------------+
input group "=== Compliance Toggles ==="
input bool   EnforceNoSpreadSlStopMoves   = true;   // MUST default ON
input bool   EnforceNoSlippageStopMoves   = true;   // MUST default ON
input bool   EnableComplianceAuditCounters= true;
//+------------------------------------------------------------------+
//| INPUTS — SYMBOL & CORE                                           |
//+------------------------------------------------------------------+
input group "=== Core Settings ==="
input string InpSymbolName                = "XAUUSD";
input long   InpMagicNumber               = 777555;
input ENUM_TIMEFRAMES InpExecTF           = PERIOD_M1;
input ENUM_TIMEFRAMES InpContextTF_M5     = PERIOD_M5;
input ENUM_TIMEFRAMES InpContextTF_M15    = PERIOD_M15;
//+------------------------------------------------------------------+
//| INPUTS — RISK PARAMETERS                                         |
//+------------------------------------------------------------------+
input group "=== Risk Parameters ==="
input double InpRiskPercent               = 0.5;    // % equity per trade
input double InpMaxDailyDD                = 3.0;    // % daily drawdown lock
input double InpMaxWeeklyDD               = 6.0;    // % weekly drawdown lock
input int    InpMaxConsecLosses           = 4;
input int    InpMaxTradesPerSession       = 5;
input double InpMaxSpreadPoints           = 50.0;   // spread entry filter
input int    InpMaxSlippagePoints         = 30;
input int    InpMaxRetries                = 3;
//+------------------------------------------------------------------+
//| INPUTS — INDICATOR PERIODS                                       |
//+------------------------------------------------------------------+
input group "=== Indicator Periods ==="
input int    InpATRPeriod                 = 14;
input int    InpADXPeriod                 = 14;
input int    InpRSIPeriod                 = 14;
input int    InpEMAFast                   = 20;
input int    InpEMASlow                   = 50;
input int    InpBBPeriod                  = 20;
input double InpBBDeviation              = 2.0;
//+------------------------------------------------------------------+
//| INPUTS — REGIME THRESHOLDS                                       |
//+------------------------------------------------------------------+
input group "=== Regime Thresholds ==="
input double InpADXTrendThreshold         = 25.0;
input double InpADXWeakThreshold          = 18.0;
input double InpBBWidthExpandThreshold    = 0.003;
input double InpBBWidthCompressThreshold  = 0.0015;
input double InpATRPercentileHigh         = 70.0;
input int    InpADXSlopeBars              = 3;
//+------------------------------------------------------------------+
//| INPUTS — ENTRY PARAMETERS                                        |
//+------------------------------------------------------------------+
input group "=== Entry Parameters ==="
input double InpRSIOverbought             = 70.0;
input double InpRSIOversold               = 30.0;
input double InpMinSetupQuality           = 0.5;
input double InpOverextensionATRMult      = 2.5;
input double InpMinBodyRatio              = 0.4;
input int    InpSessionStartHour          = 7;   // London start
input int    InpSessionEndHour            = 20;  // NY close
//+------------------------------------------------------------------+
//| INPUTS — POSITION MANAGEMENT                                     |
//+------------------------------------------------------------------+
input group "=== Position Management ==="
input double InpATRSLMultiplier           = 1.5;
input double InpInitialTPMultR            = 2.0;
input double InpBETriggerR                = 1.0;
input double InpBEBufferPoints            = 10.0;
input double InpPartial1Pct               = 30.0;  // % of original
input double InpPartial1TriggerR          = 1.0;
input double InpPartial2Pct               = 30.0;
input double InpPartial2TriggerR          = 2.0;
input int    InpMinBarsBeforeBE           = 3;
//+------------------------------------------------------------------+
//| INPUTS — TRAILING                                                |
//+------------------------------------------------------------------+
input group "=== Trailing Parameters ==="
input double InpATRTrailMultiplier        = 2.0;
input int    InpSwingLookback             = 10;
input int    InpTrailingStructureLookback = 10;
input double InpFastBankTriggerR          = 0.5;
input double InpFastBankTrailPoints       = 30.0;
//+------------------------------------------------------------------+
//| INPUTS — MDAE PARAMETERS                                         |
//+------------------------------------------------------------------+
input group "=== MDAE Parameters ==="
input int    InpSpeedWindowShort          = 5;
input int    InpSpeedWindowLong           = 20;
input int    InpVolumeAvgPeriod           = 20;
input double InpVolumePulseThreshold      = 1.5;
input int    InpTop2Lookback              = 20;
input int    InpLowVolBoxPeriod           = 10;
input double InpLowVolBoxThreshold        = 0.5;
input double InpFibLevel1                 = 1.272;
input double InpFibLevel2                 = 1.618;
input int    InpMaxPolicySwitches         = 2;
input int    InpMaxTradeAgeBars           = 120;
input double InpEntropyHighThreshold      = 0.7;
input double InpContinuationThreshold     = 0.6;
input double InpExhaustionThreshold       = 0.4;
//+------------------------------------------------------------------+
//| INPUTS — NEWS WINDOW                                             |
//+------------------------------------------------------------------+
input group "=== News Window ==="
input int    InpNewsBlockStartHour        = 13;
input int    InpNewsBlockStartMin         = 25;
input int    InpNewsBlockEndHour          = 13;
input int    InpNewsBlockEndMin           = 40;
//+------------------------------------------------------------------+
//| MANAGED STATE STRUCTURE                                          |
//+------------------------------------------------------------------+
struct ManagedState
{
   bool     active;
   ulong    ticket;
   ulong    positionId;
   int      direction;       // +1 buy, -1 sell
   double   entryPrice;
   double   initialSL;
   double   initialTP;
   double   initialVolume;
   double   riskDistance;     // price distance
   double   mfe;
   double   mae;
   double   currentR;
   bool     beApplied;
   bool     partial1Done;
   bool     partial2Done;
   ENUM_CLOSE_METHOD activeMethod;
   int      policySwitchCount;
   datetime entryTime;
   int      entryBar;
   int      barsInTrade;
   double   lastSpreadAtEntry;
   double   lastSlippageAtEntry;
   // Compliance
   int      complianceViolationCount;
   bool     stopChangedDueSpread;
   bool     stopChangedDueSlippage;
};
//+------------------------------------------------------------------+
//| GLOBAL STATE                                                     |
//+------------------------------------------------------------------+
CTrade g_trade;
// Indicator handles
int g_hATR, g_hADX, g_hRSI, g_hEMAFast, g_hEMASlow, g_hBB;
int g_hATR_M5, g_hEMAFast_M5, g_hEMASlow_M5;
int g_hATR_M15, g_hEMAFast_M15, g_hEMASlow_M15;
// Runtime state
datetime g_lastBar = 0;
double   g_startEquity = 0;
double   g_weekStartEquity = 0;
int      g_dailyLossCount = 0;
int      g_weeklyLossCount = 0;
int      g_dayStamp = 0;
int      g_weekStamp = 0;
int      g_sessionTradeCount = 0;
int      g_sessionDayStamp = 0;
// Managed trade state
ManagedState g_ms;
// Log file handle
int g_logHandle = INVALID_HANDLE;
// Compliance audit counters
int g_complianceTotalChecks = 0;
int g_complianceTotalViolations = 0;
// Regime state
ENUM_REGIME g_currentRegime = REGIME_NO_TRADE;
double g_regimeConfidence = 0.0;
string g_regimeRejectReason = "";
// Closed-trade aggregation (position-level) to avoid counting partial-close fragments as full losses
ulong  g_closedPosIds[];
double g_closedPosPnL[];
//+------------------------------------------------------------------+
//| INITIALIZATION                                                   |
//+------------------------------------------------------------------+
int OnInit()
{
   if(!EnableEA)
   {
      Print("EA is disabled via EnableEA input.");
      return INIT_SUCCEEDED;
   }
   // Setup trade object
   g_trade.SetExpertMagicNumber(InpMagicNumber);
   g_trade.SetDeviationInPoints(InpMaxSlippagePoints);
   SelectSupportedFillingMode();
   // Create indicator handles — M1
   g_hATR     = iATR(InpSymbolName, InpExecTF, InpATRPeriod);
   g_hADX     = iADX(InpSymbolName, InpExecTF, InpADXPeriod);
   g_hRSI     = iRSI(InpSymbolName, InpExecTF, InpRSIPeriod, PRICE_CLOSE);
   g_hEMAFast = iMA(InpSymbolName, InpExecTF, InpEMAFast, 0, MODE_EMA, PRICE_CLOSE);
   g_hEMASlow = iMA(InpSymbolName, InpExecTF, InpEMASlow, 0, MODE_EMA, PRICE_CLOSE);
   g_hBB      = iBands(InpSymbolName, InpExecTF, InpBBPeriod, 0, InpBBDeviation, PRICE_CLOSE);
   // Context TF handles — M5
   g_hATR_M5     = iATR(InpSymbolName, InpContextTF_M5, InpATRPeriod);
   g_hEMAFast_M5 = iMA(InpSymbolName, InpContextTF_M5, InpEMAFast, 0, MODE_EMA, PRICE_CLOSE);
   g_hEMASlow_M5 = iMA(InpSymbolName, InpContextTF_M5, InpEMASlow, 0, MODE_EMA, PRICE_CLOSE);
   // Context TF handles — M15
   g_hATR_M15     = iATR(InpSymbolName, InpContextTF_M15, InpATRPeriod);
   g_hEMAFast_M15 = iMA(InpSymbolName, InpContextTF_M15, InpEMAFast, 0, MODE_EMA, PRICE_CLOSE);
   g_hEMASlow_M15 = iMA(InpSymbolName, InpContextTF_M15, InpEMASlow, 0, MODE_EMA, PRICE_CLOSE);
   // Validate handles
   if(g_hATR==INVALID_HANDLE || g_hADX==INVALID_HANDLE || g_hRSI==INVALID_HANDLE ||
      g_hEMAFast==INVALID_HANDLE || g_hEMASlow==INVALID_HANDLE || g_hBB==INVALID_HANDLE ||
      g_hATR_M5==INVALID_HANDLE || g_hEMAFast_M5==INVALID_HANDLE || g_hEMASlow_M5==INVALID_HANDLE ||
      g_hATR_M15==INVALID_HANDLE || g_hEMAFast_M15==INVALID_HANDLE || g_hEMASlow_M15==INVALID_HANDLE)
   {
      Print("ERROR: Failed to create indicator handles!");
      return INIT_FAILED;
   }
   // Initialize equity tracking
   g_startEquity = AccountInfoDouble(ACCOUNT_EQUITY);
   g_weekStartEquity = g_startEquity;
   MqlDateTime dt;
   TimeCurrent(dt);
   g_dayStamp = dt.day_of_year;
   g_weekStamp = GetWeekStamp(dt);
   g_sessionDayStamp = g_dayStamp;
   // Reset managed state
   ResetManagedState();
   // Reconstruct state if position exists on restart
   ReconstructOpenPosition();
   // Open log file
   if(EnableLogging)
   {
      string logName = InpSymbolName + "_MDAE_Log_" + IntegerToString(InpMagicNumber) + ".csv";
      g_logHandle = FileOpen(logName, FILE_WRITE|FILE_CSV|FILE_ANSI, ',');
      if(g_logHandle != INVALID_HANDLE)
      {
         FileWrite(g_logHandle, "Timestamp","Symbol","TradeID","Event","Regime","Side",
                   "CurrentR","Method","Action","ReasonCode","Spread","Volume",
                   "Entropy","StopDueSpread","StopDueSlippage");
      }
   }
   Print("XAUUSD Adaptive MDAE EA initialized successfully.");
   return INIT_SUCCEEDED;
}
//+------------------------------------------------------------------+
//| DEINITIALIZATION                                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   // Release indicator handles
   if(g_hATR!=INVALID_HANDLE)     IndicatorRelease(g_hATR);
   if(g_hADX!=INVALID_HANDLE)     IndicatorRelease(g_hADX);
   if(g_hRSI!=INVALID_HANDLE)     IndicatorRelease(g_hRSI);
   if(g_hEMAFast!=INVALID_HANDLE) IndicatorRelease(g_hEMAFast);
   if(g_hEMASlow!=INVALID_HANDLE) IndicatorRelease(g_hEMASlow);
   if(g_hBB!=INVALID_HANDLE)      IndicatorRelease(g_hBB);
   if(g_hATR_M5!=INVALID_HANDLE)     IndicatorRelease(g_hATR_M5);
   if(g_hEMAFast_M5!=INVALID_HANDLE) IndicatorRelease(g_hEMAFast_M5);
   if(g_hEMASlow_M5!=INVALID_HANDLE) IndicatorRelease(g_hEMASlow_M5);
   if(g_hATR_M15!=INVALID_HANDLE)     IndicatorRelease(g_hATR_M15);
   if(g_hEMAFast_M15!=INVALID_HANDLE) IndicatorRelease(g_hEMAFast_M15);
   if(g_hEMASlow_M15!=INVALID_HANDLE) IndicatorRelease(g_hEMASlow_M15);
   if(g_logHandle != INVALID_HANDLE)
   {
      FileClose(g_logHandle);
      g_logHandle = INVALID_HANDLE;
   }
   Print("XAUUSD Adaptive MDAE EA deinitialized. Compliance violations: ", g_complianceTotalViolations);
}
//+------------------------------------------------------------------+
//| ONTICK                                                           |
//+------------------------------------------------------------------+
void OnTick()
{
   if(!EnableEA) return;
   // Day/Week reset
   CheckDayWeekReset();
   // 1) Manage existing position first (every tick)
   if(EnablePositionManagement && g_ms.active)
      ManagePosition();
   // 2) On new bar, evaluate entries
   if(NewBar())
   {
      if(EnableEntries)
         EvaluateEntry();
   }
}
bool IsPositionIdOpen(ulong positionId)
{
   if(positionId == 0) return false;
   for(int i = PositionsTotal()-1; i >= 0; --i)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if((ulong)PositionGetInteger(POSITION_IDENTIFIER) == positionId && PositionGetString(POSITION_SYMBOL) == InpSymbolName)
         return true;
   }
   return false;
}

int FindClosedPosIndex(ulong positionId)
{
   for(int i = 0; i < ArraySize(g_closedPosIds); i++)
      if(g_closedPosIds[i] == positionId)
         return i;
   return -1;
}

void AddClosedPosPnL(ulong positionId, double pnl)
{
   if(positionId == 0) return;
   int idx = FindClosedPosIndex(positionId);
   if(idx < 0)
   {
      int n = ArraySize(g_closedPosIds);
      if(ArrayResize(g_closedPosIds, n + 1) != n + 1) return;
      if(ArrayResize(g_closedPosPnL, n + 1) != n + 1) return;
      g_closedPosIds[n] = positionId;
      g_closedPosPnL[n] = pnl;
      return;
   }
   g_closedPosPnL[idx] += pnl;
}

double GetClosedPosPnL(ulong positionId)
{
   int idx = FindClosedPosIndex(positionId);
   if(idx < 0) return 0.0;
   return g_closedPosPnL[idx];
}

void ClearClosedPosPnL(ulong positionId)
{
   int idx = FindClosedPosIndex(positionId);
   if(idx < 0) return;
   int last = ArraySize(g_closedPosIds) - 1;
   if(last < 0) return;
   if(idx != last)
   {
      g_closedPosIds[idx] = g_closedPosIds[last];
      g_closedPosPnL[idx] = g_closedPosPnL[last];
   }
   ArrayResize(g_closedPosIds, last);
   ArrayResize(g_closedPosPnL, last);
}

//+------------------------------------------------------------------+
//| OnTradeTransaction                                               |
//+------------------------------------------------------------------+
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
   if(trans.symbol != InpSymbolName) return;
   ulong resolvedPositionId = (trans.position > 0) ? trans.position : trans.position_by;
   if(trans.type == TRADE_TRANSACTION_DEAL_ADD && trans.deal > 0 && HistoryDealSelect(trans.deal))
   {
      long dealMagic = HistoryDealGetInteger(trans.deal, DEAL_MAGIC);
      if(dealMagic != InpMagicNumber) return;
      if(resolvedPositionId == 0) resolvedPositionId = (ulong)HistoryDealGetInteger(trans.deal, DEAL_POSITION_ID);
      ENUM_DEAL_ENTRY de = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(trans.deal, DEAL_ENTRY);
      if(de == DEAL_ENTRY_OUT || de == DEAL_ENTRY_OUT_BY)
      {
         double dealPnl = HistoryDealGetDouble(trans.deal, DEAL_PROFIT) + HistoryDealGetDouble(trans.deal, DEAL_SWAP) + HistoryDealGetDouble(trans.deal, DEAL_COMMISSION);
         AddClosedPosPnL(resolvedPositionId, dealPnl);
         if(IsPositionIdOpen(resolvedPositionId))
         {
            LogDecisionEvent("TRADE_EVENT", "PARTIAL_CLOSE positionId=" + IntegerToString((int)resolvedPositionId) +
               " dealPnL=" + DoubleToString(dealPnl,2) + " aggPnL=" + DoubleToString(GetClosedPosPnL(resolvedPositionId),2), 0);
            return;
         }

         double finalProfit = GetClosedPosPnL(resolvedPositionId);
         LogDecisionEvent("TRADE_EVENT", "FULL_CLOSE positionId=" + IntegerToString((int)resolvedPositionId) +
            " finalPnL=" + DoubleToString(finalProfit,2), 0);
         LogTradeSummary("CLOSED", finalProfit);
         if(finalProfit < 0)
         {
            g_dailyLossCount++;
            g_weeklyLossCount++;
         }
         if(g_ms.active && resolvedPositionId > 0 && g_ms.positionId == resolvedPositionId)
            ResetManagedState();
         ClearClosedPosPnL(resolvedPositionId);
      }
      else if((de == DEAL_ENTRY_IN || de == DEAL_ENTRY_INOUT) && resolvedPositionId > 0)
      {
         RefreshManagedStateFromPosition(resolvedPositionId);
         double dealPrice = HistoryDealGetDouble(trans.deal, DEAL_PRICE);
         double intendedPrice = (request.price > 0.0) ? request.price : ((result.price > 0.0) ? result.price : dealPrice);
         double point = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
         if(point > 0 && intendedPrice > 0.0 && dealPrice > 0.0)
         {
            g_ms.lastSlippageAtEntry = MathAbs(dealPrice - intendedPrice) / point;
            LogDecisionEvent("ORDER_FILL", "POST_FILL_SLIPPAGE points=" + DoubleToString(g_ms.lastSlippageAtEntry,1) +
               " intended=" + DoubleToString(intendedPrice,_Digits) + " fill=" + DoubleToString(dealPrice,_Digits), 0);
         }
      }
   }
}
//+------------------------------------------------------------------+
//| HELPER — Safe Indicator Copy                                     |
//+------------------------------------------------------------------+
bool CopyBuf1(int handle, int bufIndex, int shift, double &val)
{
   double buf[1];
   if(CopyBuffer(handle, bufIndex, shift, 1, buf) != 1)
   {
      val = 0;
      return false;
   }
   val = buf[0];
   return true;
}
bool CopyBufN(int handle, int bufIndex, int shift, int count, double &arr[])
{
   if(ArrayResize(arr, count) != count) return false;
   if(CopyBuffer(handle, bufIndex, shift, count, arr) != count) return false;
   return true;
}
//+------------------------------------------------------------------+
//| HELPER — NewBar                                                  |
//+------------------------------------------------------------------+
bool NewBar()
{
   datetime t = iTime(InpSymbolName, InpExecTF, 0);
   if(t == 0) return false;
   if(t != g_lastBar)
   {
      g_lastBar = t;
      return true;
   }
   return false;
}
//+------------------------------------------------------------------+
//| HELPER — NormalizePrice                                          |
//+------------------------------------------------------------------+
double NormalizePrice(double price)
{
   double tickSize = SymbolInfoDouble(InpSymbolName, SYMBOL_TRADE_TICK_SIZE);
   if(tickSize <= 0) tickSize = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
   if(tickSize <= 0) return NormalizeDouble(price, (int)SymbolInfoInteger(InpSymbolName, SYMBOL_DIGITS));
   return NormalizeDouble(MathRound(price / tickSize) * tickSize, (int)SymbolInfoInteger(InpSymbolName, SYMBOL_DIGITS));
}
//+------------------------------------------------------------------+
//| HELPER — NormalizeLot                                            |
//+------------------------------------------------------------------+
double NormalizeLot(double lots)
{
   double minLot  = SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MIN);
   double maxLot  = SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MAX);
   double lotStep = SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_STEP);
   if(lotStep <= 0) lotStep = 0.01;
   lots = MathFloor(lots / lotStep) * lotStep;
   if(lots < minLot) lots = minLot;
   if(lots > maxLot) lots = maxLot;
   return NormalizeDouble(lots, 2);
}
//+------------------------------------------------------------------+
//| HELPER — IsOurPosition                                           |
//+------------------------------------------------------------------+
bool IsOurPosition()
{
   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != InpSymbolName) continue;
      if(PositionGetInteger(POSITION_MAGIC) != InpMagicNumber) continue;
      return true;
   }
   return false;
}
//+------------------------------------------------------------------+
//| HELPER — Get Our Position Ticket                                 |
//+------------------------------------------------------------------+
ulong GetOurTicket()
{
   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != InpSymbolName) continue;
      if(PositionGetInteger(POSITION_MAGIC) != InpMagicNumber) continue;
      return ticket;
   }
   return 0;
}
//+------------------------------------------------------------------+
//| HELPER — Current Spread                                          |
//+------------------------------------------------------------------+
double EstimateSpreadPoints()
{
   return (double)SymbolInfoInteger(InpSymbolName, SYMBOL_SPREAD);
}

int GetWeekStamp(const MqlDateTime &dt)
{
   int dow = dt.day_of_week;
   if(dow == 0) dow = 7;
   return dt.year * 1000 + (dt.day_of_year - dow + 10) / 7;
}

ulong FindManagedPositionTicket()
{
   for(int i = PositionsTotal() - 1; i >= 0; --i)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != InpSymbolName) continue;
      if(PositionGetInteger(POSITION_MAGIC) != InpMagicNumber) continue;
      return ticket;
   }
   return 0;
}

bool RefreshManagedStateFromPosition(ulong ticket)
{
   if(ticket == 0 || !PositionSelectByTicket(ticket)) return false;
   g_ms.active = true;
   g_ms.ticket = ticket;
   g_ms.positionId = (ulong)PositionGetInteger(POSITION_IDENTIFIER);
   g_ms.entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
   g_ms.direction = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) ? 1 : -1;
   if(g_ms.initialVolume <= 0.0) g_ms.initialVolume = PositionGetDouble(POSITION_VOLUME);
   return true;
}

double ComputeVWAP(int bars)
{
   double pv = 0.0, vol = 0.0;
   for(int i=0;i<bars;i++)
   {
      double h=iHigh(InpSymbolName, InpExecTF, i), l=iLow(InpSymbolName, InpExecTF, i), c=iClose(InpSymbolName, InpExecTF, i);
      double tp=(h+l+c)/3.0;
      double v=(double)iVolume(InpSymbolName, InpExecTF, i);
      pv += tp*v; vol += v;
   }
   if(vol<=0.0) return iClose(InpSymbolName, InpExecTF, 0);
   return pv/vol;
}

bool PassVWAPBias(int side, string stage)
{
   if(!EnableVWAPCheck)
   {
      LogDecisionEvent(stage, "VWAP check inactive", 0);
      return true;
   }
   double vwap = ComputeVWAP(30);
   double px = iClose(InpSymbolName, InpExecTF, 0);
   bool pass = (side > 0) ? (px >= vwap) : (px <= vwap);
   LogDecisionEvent(stage, "VWAP " + (pass ? "PASS" : "FAIL") + " px=" + DoubleToString(px,_Digits) + " vwap=" + DoubleToString(vwap,_Digits), 0);
   return pass;
}

string RetcodeToString(int rc)
{
   if(rc == TRADE_RETCODE_DONE) return "DONE";
   if(rc == TRADE_RETCODE_DONE_PARTIAL) return "DONE_PARTIAL";
   if(rc == TRADE_RETCODE_REQUOTE) return "REQUOTE";
   if(rc == TRADE_RETCODE_INVALID_FILL) return "INVALID_FILL";
   if(rc == TRADE_RETCODE_REJECT) return "REJECT";
   return "RET_" + IntegerToString(rc);
}

void SelectSupportedFillingMode()
{
   long fillMask = SymbolInfoInteger(InpSymbolName, SYMBOL_FILLING_MODE);
   ENUM_ORDER_TYPE_FILLING selected = ORDER_FILLING_RETURN;
   if((fillMask & ORDER_FILLING_FOK) == ORDER_FILLING_FOK) selected = ORDER_FILLING_FOK;
   else if((fillMask & ORDER_FILLING_IOC) == ORDER_FILLING_IOC) selected = ORDER_FILLING_IOC;
   else selected = ORDER_FILLING_RETURN;
   g_trade.SetTypeFilling(selected);
   Print("Selected filling mode=", (int)selected, " mask=", (int)fillMask);
}

//+------------------------------------------------------------------+
//| HELPER — Day/Week Reset                                          |
//+------------------------------------------------------------------+
void CheckDayWeekReset()
{
   MqlDateTime dt;
   TimeCurrent(dt);
   int curDay = dt.day_of_year;
   int curWeek = GetWeekStamp(dt);
   int oldDay = g_dayStamp;
   int oldWeek = g_weekStamp;
   if(curDay != g_dayStamp)
   {
      g_dayStamp = curDay;
      g_startEquity = AccountInfoDouble(ACCOUNT_EQUITY);
      g_dailyLossCount = 0;
      LogDecisionEvent("RESET", "DAY reset " + IntegerToString(oldDay) + " -> " + IntegerToString(curDay), 0);
   }
   if(curDay != g_sessionDayStamp)
   {
      g_sessionDayStamp = curDay;
      g_sessionTradeCount = 0;
   }
   if(curWeek != g_weekStamp)
   {
      g_weekStamp = curWeek;
      g_weekStartEquity = AccountInfoDouble(ACCOUNT_EQUITY);
      g_weeklyLossCount = 0;
      LogDecisionEvent("RESET", "WEEK reset " + IntegerToString(oldWeek) + " -> " + IntegerToString(curWeek), 0);
   }
}
//+------------------------------------------------------------------+
//| HELPER — Reset Managed State                                     |
//+------------------------------------------------------------------+
void ResetManagedState()
{
   ZeroMemory(g_ms);
   g_ms.active = false;
   g_ms.activeMethod = CM_FIXED;
}
//+------------------------------------------------------------------+
//| HELPER — Reconstruct Open Position on Restart                    |
//+------------------------------------------------------------------+
void ReconstructOpenPosition()
{
   ulong ticket = GetOurTicket();
   if(ticket == 0) return;
   if(!PositionSelectByTicket(ticket)) return;
   g_ms.active        = true;
   g_ms.ticket        = ticket;
   g_ms.positionId    = (ulong)PositionGetInteger(POSITION_IDENTIFIER);
   g_ms.entryPrice    = PositionGetDouble(POSITION_PRICE_OPEN);
   g_ms.initialSL     = PositionGetDouble(POSITION_SL);
   g_ms.initialTP     = PositionGetDouble(POSITION_TP);
   g_ms.initialVolume = PositionGetDouble(POSITION_VOLUME);
   g_ms.entryTime     = (datetime)PositionGetInteger(POSITION_TIME);
   ENUM_POSITION_TYPE ptype = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
   g_ms.direction = (ptype == POSITION_TYPE_BUY) ? 1 : -1;
   g_ms.riskDistance = MathAbs(g_ms.entryPrice - g_ms.initialSL);
   if(g_ms.riskDistance <= 0)
   {
      double atr = 0;
      CopyBuf1(g_hATR, 0, 1, atr);
      g_ms.riskDistance = (atr > 0) ? atr * InpATRSLMultiplier : 100 * SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
   }
   g_ms.mfe = 0;
   g_ms.mae = 0;
   g_ms.beApplied = false;
   g_ms.partial1Done = false;
   g_ms.partial2Done = false;
   g_ms.activeMethod = CM_FIXED;
   g_ms.policySwitchCount = 0;
   g_ms.entryBar = 0;
   g_ms.barsInTrade = 0;
   g_ms.complianceViolationCount = 0;
   g_ms.stopChangedDueSpread = false;
   g_ms.stopChangedDueSlippage = false;
   Print("INFO: Reconstructed open position ticket=", ticket, " dir=", g_ms.direction, " entry=", g_ms.entryPrice);
   LogDecisionEvent("RECONSTRUCTION", "Reconstructed existing position on restart", 0);
}
//+------------------------------------------------------------------+
//| HELPER — Session Filter                                          |
//+------------------------------------------------------------------+
bool IsWithinSession()
{
   if(!EnableSessionFilter) return true;
   MqlDateTime dt;
   TimeCurrent(dt);
   int hour = dt.hour;
   return (hour >= InpSessionStartHour && hour < InpSessionEndHour);
}
//+------------------------------------------------------------------+
//| HELPER — News Window Block                                       |
//+------------------------------------------------------------------+
bool IsNewsBlocked()
{
   if(!EnableNewsWindowBlock) return false;
   MqlDateTime dt;
   TimeCurrent(dt);
   int nowMin = dt.hour * 60 + dt.min;
   int startMin = InpNewsBlockStartHour * 60 + InpNewsBlockStartMin;
   int endMin   = InpNewsBlockEndHour * 60 + InpNewsBlockEndMin;
   return (nowMin >= startMin && nowMin <= endMin);
}
//+------------------------------------------------------------------+
//| RISK GOVERNOR — IsTradingLocked                                  |
//+------------------------------------------------------------------+
ENUM_RISK_STATE RiskCheck()
{
   if(!EnableRiskGovernor) return RISK_ALLOW;
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   // Daily DD lock
   if(EnableDailyDDLock && g_startEquity > 0)
   {
      double ddPct = ((g_startEquity - equity) / g_startEquity) * 100.0;
      if(ddPct >= InpMaxDailyDD)
      {
         LogDecisionEvent("RISK_GOVERNOR", "Daily DD lock triggered: " + DoubleToString(ddPct,2) + "%", 0);
         return RISK_BLOCK;
      }
   }
   // Weekly DD lock
   if(EnableWeeklyDDLock && g_weekStartEquity > 0)
   {
      double wddPct = ((g_weekStartEquity - equity) / g_weekStartEquity) * 100.0;
      if(wddPct >= InpMaxWeeklyDD)
      {
         LogDecisionEvent("RISK_GOVERNOR", "Weekly DD lock triggered: " + DoubleToString(wddPct,2) + "%", 0);
         return RISK_BLOCK;
      }
   }
   // Consecutive loss lock
   if(EnableConsecutiveLossLock && g_dailyLossCount >= InpMaxConsecLosses)
   {
      LogDecisionEvent("RISK_GOVERNOR", "Consecutive loss lock: " + IntegerToString(g_dailyLossCount), 0);
      return RISK_BLOCK;
   }
   // Max trades per session
   if(EnableMaxTradesPerSession && g_sessionTradeCount >= InpMaxTradesPerSession)
   {
      LogDecisionEvent("RISK_GOVERNOR", "Max session trades reached", 0);
      return RISK_BLOCK;
   }
   // Reduce risk if approaching limits
   if(EnableDailyDDLock && g_startEquity > 0)
   {
      double ddPct = ((g_startEquity - equity) / g_startEquity) * 100.0;
      if(ddPct >= InpMaxDailyDD * 0.7)
         return RISK_REDUCE;
   }
   return RISK_ALLOW;
}
bool IsTradingLocked()
{
   return (RiskCheck() == RISK_BLOCK);
}
//+------------------------------------------------------------------+
//| RISK GOVERNOR — CalcLotSizeByRisk                                |
//+------------------------------------------------------------------+
double CalcLotSizeByRisk(double slDistancePrice)
{
   if(slDistancePrice <= 0) return SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MIN);
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   double riskPct = InpRiskPercent;
   // Reduce if risk state is REDUCE
   ENUM_RISK_STATE rs = RiskCheck();
   if(rs == RISK_REDUCE) riskPct *= 0.5;
   double riskMoney = equity * riskPct / 100.0;
   double tickValue = SymbolInfoDouble(InpSymbolName, SYMBOL_TRADE_TICK_VALUE);
   double tickSize  = SymbolInfoDouble(InpSymbolName, SYMBOL_TRADE_TICK_SIZE);
   if(tickValue <= 0 || tickSize <= 0)
      return SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MIN);
   double slTicks = slDistancePrice / tickSize;
   double lossPerLot = slTicks * tickValue;
   if(lossPerLot <= 0) return SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MIN);
   double lots = riskMoney / lossPerLot;
   return NormalizeLot(lots);
}
//+------------------------------------------------------------------+
//| REGIME ENGINE — DetectRegime                                     |
//+------------------------------------------------------------------+
ENUM_REGIME DetectRegime()
{
   if(!EnableRegimeEngine) return REGIME_TREND_EXPANSION; // default if disabled
   double adxVal = 0, adxPrev = 0;
   double bbUpper = 0, bbLower = 0, bbMiddle = 0;
   double close0 = 0;
   if(!CopyBuf1(g_hADX, 0, 1, adxVal)) { g_regimeRejectReason = "ADX_COPY_FAIL"; return REGIME_NO_TRADE; }
   if(!CopyBuf1(g_hADX, 0, 1 + InpADXSlopeBars, adxPrev)) { g_regimeRejectReason = "ADX_SLOPE_FAIL"; return REGIME_NO_TRADE; }
   if(!CopyBuf1(g_hBB, 1, 1, bbUpper)) { g_regimeRejectReason = "BB_COPY_FAIL"; return REGIME_NO_TRADE; }
   if(!CopyBuf1(g_hBB, 2, 1, bbLower)) { g_regimeRejectReason = "BB_COPY_FAIL"; return REGIME_NO_TRADE; }
   if(!CopyBuf1(g_hBB, 0, 1, bbMiddle)) { g_regimeRejectReason = "BB_COPY_FAIL"; return REGIME_NO_TRADE; }
   close0 = iClose(InpSymbolName, InpExecTF, 1);
   if(close0 <= 0) { g_regimeRejectReason = "PRICE_FAIL"; return REGIME_NO_TRADE; }
   double bbWidth = (bbMiddle > 0) ? (bbUpper - bbLower) / bbMiddle : 0;
   double adxSlope = adxVal - adxPrev;
   // ATR percentile estimation
   double atrArr[];
   double atrNow = 0;
   CopyBuf1(g_hATR, 0, 1, atrNow);
   double atrPercentile = 50.0; // default
   if(CopyBufN(g_hATR, 0, 1, 100, atrArr))
   {
      int countBelow = 0;
      for(int i = 0; i < ArraySize(atrArr); i++)
         if(atrArr[i] <= atrNow) countBelow++;
      atrPercentile = (double)countBelow / (double)ArraySize(atrArr) * 100.0;
   }
   // Classification
   bool trendSignal = (adxVal >= InpADXTrendThreshold && adxSlope > 0 && bbWidth >= InpBBWidthExpandThreshold && atrPercentile >= InpATRPercentileHigh);
   bool rangeSignal = (adxVal <= InpADXWeakThreshold && bbWidth <= InpBBWidthCompressThreshold);
   g_regimeConfidence = 0.0;
   if(trendSignal)
   {
      g_regimeConfidence = MathMin(1.0, (adxVal / 50.0) * (bbWidth / InpBBWidthExpandThreshold));
      g_regimeRejectReason = "";
      g_currentRegime = REGIME_TREND_EXPANSION;
      LogDecisionEvent("REGIME", "TREND_EXPANSION ADX=" + DoubleToString(adxVal,1) + " BBW=" + DoubleToString(bbWidth,5) + " ATRPct=" + DoubleToString(atrPercentile,1), g_regimeConfidence);
      return REGIME_TREND_EXPANSION;
   }
   else if(rangeSignal)
   {
      g_regimeConfidence = MathMin(1.0, (1.0 - adxVal / InpADXTrendThreshold) * (1.0 - bbWidth / InpBBWidthExpandThreshold));
      g_regimeRejectReason = "";
      g_currentRegime = REGIME_MEAN_REVERSION;
      LogDecisionEvent("REGIME", "MEAN_REVERSION ADX=" + DoubleToString(adxVal,1) + " BBW=" + DoubleToString(bbWidth,5) + " ATRPct=" + DoubleToString(atrPercentile,1), g_regimeConfidence);
      return REGIME_MEAN_REVERSION;
   }
   else
   {
      if(!EnableNoTradeState)
      {
         // Fall back to trend if NO_TRADE disabled
         g_currentRegime = REGIME_TREND_EXPANSION;
         return REGIME_TREND_EXPANSION;
      }
      g_regimeRejectReason = "UNCERTAIN";
      g_currentRegime = REGIME_NO_TRADE;
      LogDecisionEvent("REGIME", "NO_TRADE ADX=" + DoubleToString(adxVal,1) + " BBW=" + DoubleToString(bbWidth,5) + " ATRPct=" + DoubleToString(atrPercentile,1), 0);
      return REGIME_NO_TRADE;
   }
}
//+------------------------------------------------------------------+
//| MTF CONTEXT CHECK                                                |
//+------------------------------------------------------------------+
int GetMTFDirection()
{
   if(!EnableMTFContextCheck) return 0; // no filter
   double emaFast5=0, emaSlow5=0, emaFast15=0, emaSlow15=0;
   if(!CopyBuf1(g_hEMAFast_M5, 0, 0, emaFast5)) return 0;
   if(!CopyBuf1(g_hEMASlow_M5, 0, 0, emaSlow5)) return 0;
   if(!CopyBuf1(g_hEMAFast_M15, 0, 0, emaFast15)) return 0;
   if(!CopyBuf1(g_hEMASlow_M15, 0, 0, emaSlow15)) return 0;
   int dir5 = (emaFast5 > emaSlow5) ? 1 : -1;
   int dir15 = (emaFast15 > emaSlow15) ? 1 : -1;
   if(dir5 == dir15) return dir5;
   return 0; // conflict
}
//+------------------------------------------------------------------+
//| SETUP ENGINE — EvaluateTrendEntry                                |
//+------------------------------------------------------------------+
int EvaluateTrendEntry(double &quality)
{
   quality = 0;
   if(!EnableTrendModule) return 0;
   double emaFast=0, emaSlow=0, rsi=0, atr=0;
   if(!CopyBuf1(g_hEMAFast, 0, 1, emaFast)) return 0;
   if(!CopyBuf1(g_hEMASlow, 0, 1, emaSlow)) return 0;
   if(!CopyBuf1(g_hRSI, 0, 1, rsi)) return 0;
   if(!CopyBuf1(g_hATR, 0, 1, atr)) return 0;
   double close0 = iClose(InpSymbolName, InpExecTF, 1);
   double close1 = iClose(InpSymbolName, InpExecTF, 2);
   double close2 = iClose(InpSymbolName, InpExecTF, 3);
   double open0  = iOpen(InpSymbolName, InpExecTF, 1);
   if(close0 <= 0 || close1 <= 0 || atr <= 0) return 0;
   // Overextension filter
   double distFromEMA = MathAbs(close0 - emaFast);
   if(distFromEMA > InpOverextensionATRMult * atr) return 0;
   // Body ratio filter
   double bodySize = MathAbs(close0 - open0);
   double candleRange = iHigh(InpSymbolName, InpExecTF, 1) - iLow(InpSymbolName, InpExecTF, 1);
   if(candleRange > 0 && (bodySize / candleRange) < InpMinBodyRatio) return 0;
   // MTF alignment
   int mtfDir = GetMTFDirection();
   // BUY setup
   if(emaFast > emaSlow && close0 > emaFast && rsi > 40 && rsi < InpRSIOverbought)
   {
      // Pullback recovery: previous bar was near or below EMA, current reclaimed
      bool pullback = (close1 <= emaFast * 1.001 || close2 <= emaFast * 1.001);
      if(!pullback) pullback = (iLow(InpSymbolName, InpExecTF, 2) <= emaFast);
      if(pullback)
      {
         if(mtfDir != 0 && mtfDir != 1) return 0; // MTF conflict
         quality = 0.5 + (rsi - 40) / 60.0 * 0.3 + g_regimeConfidence * 0.2;
         quality = MathMin(1.0, quality);
         if(!PassVWAPBias(1, "ENTRY_EVAL")) return 0;
         LogDecisionEvent("ENTRY_EVAL", "TREND_BUY quality=" + DoubleToString(quality,2), quality);
         return 1;
      }
   }
   // SELL setup
   if(emaFast < emaSlow && close0 < emaFast && rsi < 60 && rsi > InpRSIOversold)
   {
      bool pullback = (close1 >= emaFast * 0.999 || close2 >= emaFast * 0.999);
      if(!pullback) pullback = (iHigh(InpSymbolName, InpExecTF, 2) >= emaFast);
      if(pullback)
      {
         if(mtfDir != 0 && mtfDir != -1) return 0;
         quality = 0.5 + (60 - rsi) / 60.0 * 0.3 + g_regimeConfidence * 0.2;
         quality = MathMin(1.0, quality);
         if(!PassVWAPBias(-1, "ENTRY_EVAL")) return 0;
         LogDecisionEvent("ENTRY_EVAL", "TREND_SELL quality=" + DoubleToString(quality,2), quality);
         return -1;
      }
   }
   return 0;
}
//+------------------------------------------------------------------+
//| SETUP ENGINE — EvaluateMeanReversionEntry                        |
//+------------------------------------------------------------------+
int EvaluateMeanReversionEntry(double &quality)
{
   quality = 0;
   if(!EnableMeanReversionModule) return 0;
   double bbUpper=0, bbLower=0, bbMiddle=0, rsi=0, atr=0, adxVal=0;
   if(!CopyBuf1(g_hBB, 1, 1, bbUpper)) return 0;
   if(!CopyBuf1(g_hBB, 2, 1, bbLower)) return 0;
   if(!CopyBuf1(g_hBB, 0, 1, bbMiddle)) return 0;
   if(!CopyBuf1(g_hRSI, 0, 1, rsi)) return 0;
   if(!CopyBuf1(g_hATR, 0, 1, atr)) return 0;
   if(!CopyBuf1(g_hADX, 0, 1, adxVal)) return 0;
   double close0 = iClose(InpSymbolName, InpExecTF, 1);
   double close1 = iClose(InpSymbolName, InpExecTF, 2);
   double open0  = iOpen(InpSymbolName, InpExecTF, 1);
   double low0   = iLow(InpSymbolName, InpExecTF, 1);
   double high0  = iHigh(InpSymbolName, InpExecTF, 1);
   if(close0 <= 0 || atr <= 0) return 0;
   // Anti-fade filter: reject if ADX rising sharply (breakout pressure)
   double adxPrev = 0;
   CopyBuf1(g_hADX, 0, 4, adxPrev);
   if(adxVal - adxPrev > 5.0) return 0; // strong breakout building, don't fade
   // Body ratio
   double bodySize = MathAbs(close0 - open0);
   double candleRange = high0 - low0;
   int mtfDir = GetMTFDirection();
   // BUY (lower band bounce)
   if(low0 <= bbLower && close0 > bbLower && rsi < InpRSIOversold + 10)
   {
      // Rejection candle: close in upper half
      double clv = (candleRange > 0) ? (close0 - low0) / candleRange : 0.5;
      if(clv < 0.4) return 0; // weak rejection
      if(mtfDir != 0 && mtfDir != 1) return 0;
      quality = 0.5 + (InpRSIOversold + 10 - rsi) / 40.0 * 0.3 + clv * 0.2;
      quality = MathMin(1.0, quality);
      if(!PassVWAPBias(1, "ENTRY_EVAL")) return 0;
      LogDecisionEvent("ENTRY_EVAL", "MR_BUY quality=" + DoubleToString(quality,2), quality);
      return 1;
   }
   // SELL (upper band rejection)
   if(high0 >= bbUpper && close0 < bbUpper && rsi > InpRSIOverbought - 10)
   {
      double clv = (candleRange > 0) ? (high0 - close0) / candleRange : 0.5;
      if(clv < 0.4) return 0;
      if(mtfDir != 0 && mtfDir != -1) return 0;
      quality = 0.5 + (rsi - (InpRSIOverbought - 10)) / 40.0 * 0.3 + clv * 0.2;
      quality = MathMin(1.0, quality);
      if(!PassVWAPBias(-1, "ENTRY_EVAL")) return 0;
      LogDecisionEvent("ENTRY_EVAL", "MR_SELL quality=" + DoubleToString(quality,2), quality);
      return -1;
   }
   return 0;
}
//+------------------------------------------------------------------+
//| ENTRY EVALUATION — Main                                          |
//+------------------------------------------------------------------+
void EvaluateEntry()
{
   // Pre-checks
   if(IsTradingLocked())
   {
      LogDecisionEvent("PRECHECK", "Trading locked by RiskGovernor", 0);
      return;
   }
   if(!IsWithinSession())
   {
      if(EnableDebugPrints) Print("DEBUG: Outside trading session");
      return;
   }
   if(IsNewsBlocked())
   {
      LogDecisionEvent("PRECHECK", "News window block active", 0);
      return;
   }
   // Spread entry filter
   if(EnableSpreadEntryFilter)
   {
      double spread = EstimateSpreadPoints();
      if(spread > InpMaxSpreadPoints)
      {
         LogDecisionEvent("PRECHECK", "Spread too high: " + DoubleToString(spread,1), 0);
         return;
      }
   }
   // Signals are computed from closed candles (shift=1) to avoid look-ahead / unstable-bar noise.
   if(EnableSlippageEntryFilter)
   {
      static double s_prevMid = 0;
      double askPx = SymbolInfoDouble(InpSymbolName, SYMBOL_ASK);
      double bidPx = SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
      double point = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
      if(askPx > 0 && bidPx > 0 && point > 0)
      {
         double mid = (askPx + bidPx) * 0.5;
         double tickJumpPts = (s_prevMid > 0) ? MathAbs(mid - s_prevMid) / point : 0;
         s_prevMid = mid;
         if(tickJumpPts > InpMaxSlippagePoints)
         {
            LogDecisionEvent("PRECHECK", "Slippage proxy too high: tick jump=" + DoubleToString(tickJumpPts,1), 0);
            return;
         }
      }
   }
   if(EnableOnePositionOnly)
   {
      LogDecisionEvent("PRECHECK", "One-position policy active", 0);
      if(IsOurPosition())
      {
         LogDecisionEvent("PRECHECK", "One-position policy active; existing managed position present", 0);
         return;
      }
   }
   else
   {
      LogDecisionEvent("PRECHECK", "One-position policy inactive; multi-entry allowed", 0);
   }
   // Detect regime
   ENUM_REGIME regime = DetectRegime();
   if(regime == REGIME_NO_TRADE) return;
   // Evaluate setup based on regime
   double quality = 0;
   int side = 0;
   if(regime == REGIME_TREND_EXPANSION)
   {
      side = EvaluateTrendEntry(quality);
   }
   else if(regime == REGIME_MEAN_REVERSION)
   {
      side = EvaluateMeanReversionEntry(quality);
   }
   if(side == 0) return;
   if(quality < InpMinSetupQuality)
   {
      LogDecisionEvent("ENTRY_DECISION", "Rejected: quality " + DoubleToString(quality,2) + " < threshold", quality);
      return;
   }
   // Open trade
   OpenTrade(side, regime);
}
//+------------------------------------------------------------------+
//| OPEN TRADE                                                       |
//+------------------------------------------------------------------+
void OpenTrade(int side, ENUM_REGIME regime)
{
   double atr = 0;
   if(!CopyBuf1(g_hATR, 0, 1, atr) || atr <= 0)
   {
      LogDecisionEvent("ORDER_SUBMIT", "ATR read fail, cannot compute SL", 0);
      return;
   }
   double ask = SymbolInfoDouble(InpSymbolName, SYMBOL_ASK);
   double bid = SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
   if(ask <= 0 || bid <= 0) return;
   double slDist = atr * InpATRSLMultiplier;
   double tpDist = slDist * InpInitialTPMultR;
   double entryPrice, sl, tp;
   ENUM_ORDER_TYPE orderType;
   if(side == 1)
   {
      entryPrice = ask;
      sl = NormalizePrice(entryPrice - slDist);
      tp = NormalizePrice(entryPrice + tpDist);
      orderType = ORDER_TYPE_BUY;
   }
   else
   {
      entryPrice = bid;
      sl = NormalizePrice(entryPrice + slDist);
      tp = NormalizePrice(entryPrice - tpDist);
      orderType = ORDER_TYPE_SELL;
   }
   // Validate stop distance vs broker minimum
   long stopsLevel = SymbolInfoInteger(InpSymbolName, SYMBOL_TRADE_STOPS_LEVEL);
   double minStopDist = stopsLevel * SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
   if(slDist < minStopDist)
   {
      slDist = minStopDist * 1.1;
      if(side == 1) sl = NormalizePrice(entryPrice - slDist);
      else          sl = NormalizePrice(entryPrice + slDist);
   }
   double lots = CalcLotSizeByRisk(slDist);
   // Margin check
   double marginRequired = 0;
   if(!OrderCalcMargin(orderType, InpSymbolName, lots, entryPrice, marginRequired))
   {
      LogDecisionEvent("ORDER_SUBMIT", "Margin calc fail", 0);
      return;
   }
   if(marginRequired > AccountInfoDouble(ACCOUNT_MARGIN_FREE) * 0.9)
   {
      LogDecisionEvent("ORDER_SUBMIT", "Insufficient margin", 0);
      return;
   }
   // Send order with retry
   bool success = false;
   double spreadAtEntry = EstimateSpreadPoints();
   for(int retry = 0; retry < InpMaxRetries; retry++)
   {
      double requestedPrice = (side == 1) ? SymbolInfoDouble(InpSymbolName, SYMBOL_ASK) : SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
      if(side == 1)
         success = g_trade.Buy(lots, InpSymbolName, requestedPrice, sl, tp, "MDAE_" + EnumToString(regime));
      else
         success = g_trade.Sell(lots, InpSymbolName, requestedPrice, sl, tp, "MDAE_" + EnumToString(regime));
      if(success)
      {
         double fillPrice = g_trade.ResultPrice();
         if(fillPrice <= 0) fillPrice = requestedPrice;
         double point = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
         double fillSlippagePts = (point > 0 && requestedPrice > 0) ? MathAbs(fillPrice - requestedPrice) / point : 0;
         if(EnableSlippageEntryFilter && fillSlippagePts > InpMaxSlippagePoints)
         {
            LogDecisionEvent("ORDER_SUBMIT", "Fill slippage " + DoubleToString(fillSlippagePts,1) + " > max " + IntegerToString(InpMaxSlippagePoints) + ", closing and retrying", 0);
            ulong slippedTicket = FindManagedPositionTicket();
            if(slippedTicket > 0)
               g_trade.PositionClose(slippedTicket);
            success = false;
            Sleep(150);
            continue;
         }
         ulong resultTicket = FindManagedPositionTicket();
         if(resultTicket > 0)
         {
            if(!PositionSelectByTicket(resultTicket))
            {
               LogDecisionEvent("ORDER_SUBMIT", "PositionSelectByTicket failed for ticket=" + IntegerToString((int)resultTicket) + ", using refresh fallback", 0);
               if(!RefreshManagedStateFromPosition(resultTicket))
               {
                  LogDecisionEvent("ORDER_SUBMIT", "RefreshManagedStateFromPosition fallback failed for ticket=" + IntegerToString((int)resultTicket), 0);
                  return;
               }
            }
            g_ms.active = true;
            g_ms.ticket = resultTicket;
            g_ms.positionId = (ulong)PositionGetInteger(POSITION_IDENTIFIER);
            g_ms.entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
            g_ms.direction = side;
            g_ms.initialSL = sl;
            g_ms.initialTP = tp;
            g_ms.initialVolume = lots;
            g_ms.riskDistance = slDist;
            g_ms.mfe = 0;
            g_ms.mae = 0;
            g_ms.currentR = 0;
            g_ms.beApplied = false;
            g_ms.partial1Done = false;
            g_ms.partial2Done = false;
            g_ms.activeMethod = (regime == REGIME_MEAN_REVERSION) ? CM_FIXED : CM_ATR;
            g_ms.policySwitchCount = 0;
            g_ms.entryTime = TimeCurrent();
            g_ms.entryBar = 0;
            g_ms.barsInTrade = 0;
            g_ms.lastSpreadAtEntry = spreadAtEntry;
            g_ms.lastSlippageAtEntry = 0;
            g_ms.complianceViolationCount = 0;
            g_ms.stopChangedDueSpread = false;
            g_ms.stopChangedDueSlippage = false;
            g_sessionTradeCount++;
            LogDecisionEvent("ORDER_SUBMIT", "Opened " + ((side==1)?"BUY":"SELL") +
               " lots=" + DoubleToString(lots,2) +
               " SL=" + DoubleToString(sl, (int)SymbolInfoInteger(InpSymbolName, SYMBOL_DIGITS)) +
               " TP=" + DoubleToString(tp, (int)SymbolInfoInteger(InpSymbolName, SYMBOL_DIGITS)) +
               " spread=" + DoubleToString(spreadAtEntry,1), 0);
         }
         break;
      }
      else
      {
         int err = (int)g_trade.ResultRetcode();
         if(EnableDebugPrints) Print("DEBUG: Order send retry ", retry, " err=", err);
         if(err == TRADE_RETCODE_INVALID_FILL)
         {
            LogDecisionEvent("ORDER_SUBMIT", "Invalid fill mode; re-selecting supported mode", 0);
            SelectSupportedFillingMode();
            Sleep(100);
         }
         else if(err == 10004 || err == 10006 || err == 10013) // requote / reject / invalid price
            Sleep(200);
         else
            break;
      }
   }
   if(!success)
   {
      LogDecisionEvent("ORDER_SUBMIT", "FAILED retcode=" + IntegerToString(g_trade.ResultRetcode()), 0);
   }
}
//+------------------------------------------------------------------+
//| MDAE FEATURES — ComputePipsPerSecond                             |
//+------------------------------------------------------------------+
struct MDAEFeatures
{
   double speedNow;
   double speedShort;
   double speedLong;
   double acceleration;
   double volumePulse;
   double bodyExpansion;
   bool   top2Breakout;
   bool   inLowVolBox;
   double fibProximity;
   double entropyScore;
   double continuationScore;
};
void ComputeMDAEFeatures(MDAEFeatures &feat)
{
   ZeroMemory(feat);
   double close0 = iClose(InpSymbolName, InpExecTF, 0);
   if(close0 <= 0) return;
   // Speed: pips per second
   if(EnablePipsPerSecond)
   {
      double close1 = iClose(InpSymbolName, InpExecTF, 1);
      datetime time0 = iTime(InpSymbolName, InpExecTF, 0);
      datetime time1 = iTime(InpSymbolName, InpExecTF, 1);
      double dtSec = (double)(time0 - time1);
      if(dtSec > 0)
         feat.speedNow = MathAbs(close0 - close1) / dtSec;
      // Short baseline
      double closeS = iClose(InpSymbolName, InpExecTF, InpSpeedWindowShort);
      datetime timeS = iTime(InpSymbolName, InpExecTF, InpSpeedWindowShort);
      double dtS = (double)(time0 - timeS);
      if(dtS > 0)
         feat.speedShort = MathAbs(close0 - closeS) / dtS;
      // Long baseline
      double closeL = iClose(InpSymbolName, InpExecTF, InpSpeedWindowLong);
      datetime timeL = iTime(InpSymbolName, InpExecTF, InpSpeedWindowLong);
      double dtL = (double)(time0 - timeL);
      if(dtL > 0)
         feat.speedLong = MathAbs(close0 - closeL) / dtL;
      feat.acceleration = (feat.speedLong > 0) ? feat.speedShort / feat.speedLong : 1.0;
   }
   // Volume pulse
   if(EnableVolumePulse)
   {
      long vol0 = iVolume(InpSymbolName, InpExecTF, 0);
      double volAvg = 0;
      for(int i = 1; i <= InpVolumeAvgPeriod; i++)
         volAvg += (double)iVolume(InpSymbolName, InpExecTF, i);
      volAvg /= InpVolumeAvgPeriod;
      feat.volumePulse = (volAvg > 0) ? (double)vol0 / volAvg : 1.0;
   }
   // Body expansion
   if(EnableBodyExpansion)
   {
      double body0 = MathAbs(iClose(InpSymbolName, InpExecTF, 0) - iOpen(InpSymbolName, InpExecTF, 0));
      double body1 = MathAbs(iClose(InpSymbolName, InpExecTF, 1) - iOpen(InpSymbolName, InpExecTF, 1));
      feat.bodyExpansion = (body1 > 0) ? body0 / body1 : 1.0;
   }
   // Top-2 breakout
   if(EnableTop2BreakoutCheck)
   {
      double highest = close0, secondHighest = close0;
      double lowest = close0, secondLowest = close0;
      for(int i = 1; i <= InpTop2Lookback; i++)
      {
         double h = iHigh(InpSymbolName, InpExecTF, i);
         double l = iLow(InpSymbolName, InpExecTF, i);
         if(h > highest) { secondHighest = highest; highest = h; }
         else if(h > secondHighest) secondHighest = h;
         if(l < lowest) { secondLowest = lowest; lowest = l; }
         else if(l < secondLowest) secondLowest = l;
      }
      if(g_ms.direction == 1)
         feat.top2Breakout = (close0 > secondHighest);
      else
         feat.top2Breakout = (close0 < secondLowest);
   }
   // Low volume box
   if(EnableLowVolumeBox)
   {
      double volAvg = 0;
      for(int i = 0; i < InpLowVolBoxPeriod; i++)
         volAvg += (double)iVolume(InpSymbolName, InpExecTF, i);
      volAvg /= InpLowVolBoxPeriod;
      double longVolAvg = 0;
      for(int i = 0; i < InpVolumeAvgPeriod; i++)
         longVolAvg += (double)iVolume(InpSymbolName, InpExecTF, i);
      longVolAvg /= InpVolumeAvgPeriod;
      feat.inLowVolBox = (longVolAvg > 0 && volAvg / longVolAvg < InpLowVolBoxThreshold);
   }
   // Fib guidance
   if(EnableFibGuidance && g_ms.active && g_ms.riskDistance > 0)
   {
      double progress = g_ms.currentR;
      feat.fibProximity = 0;
      if(MathAbs(progress - InpFibLevel1) < 0.1) feat.fibProximity = 0.7;
      if(MathAbs(progress - InpFibLevel2) < 0.1) feat.fibProximity = 1.0;
   }
   // Entropy scoring
   if(EnableEntropyScoring)
   {
      // Composite uncertainty: high when signals conflict
      double speedSignal = (feat.acceleration > 1.2) ? 1.0 : (feat.acceleration < 0.8) ? -1.0 : 0.0;
      double volSignal   = (feat.volumePulse > InpVolumePulseThreshold) ? 1.0 : (feat.volumePulse < 0.7) ? -1.0 : 0.0;
      double structSignal = feat.top2Breakout ? 1.0 : 0.0;
      double boxSignal   = feat.inLowVolBox ? -1.0 : 0.0;
      double sum = speedSignal + volSignal + structSignal + boxSignal;
      double maxAbs = 4.0;
      double agreement = MathAbs(sum) / maxAbs;
      feat.entropyScore = 1.0 - agreement; // high entropy = disagreement
      // Continuation score
      feat.continuationScore = (sum > 0 && g_ms.direction == 1) || (sum < 0 && g_ms.direction == -1) ?
                               agreement : (1.0 - agreement) * 0.3;
   }
}
//+------------------------------------------------------------------+
//| MDAE — SelectCloseMethod                                         |
//+------------------------------------------------------------------+
ENUM_CLOSE_METHOD SelectCloseMethod(const MDAEFeatures &feat)
{
   if(!EnableMDAE) return CM_FIXED;
   // Invalidation / thesis break
   if(feat.entropyScore > InpEntropyHighThreshold && feat.continuationScore < InpExhaustionThreshold)
   {
      return CM_EXIT;
   }
   // Fast bank mode
   if(EnableFastBankMode && feat.inLowVolBox && g_ms.currentR > InpFastBankTriggerR)
   {
      return CM_FAST;
   }
   // Trend continuation: speed + volume + breakout
   if(feat.continuationScore > InpContinuationThreshold)
   {
      if(EnableSwingTrail && feat.top2Breakout && feat.volumePulse > InpVolumePulseThreshold)
         return CM_SWING;
      if(EnableATRTrail)
         return CM_ATR;
   }
   // Hybrid: mixed signals
   if(EnableHybridCloseMode && feat.entropyScore > 0.4 && feat.entropyScore <= InpEntropyHighThreshold)
   {
      return CM_HYBRID;
   }
   // Default fixed
   return CM_FIXED;
}
//+------------------------------------------------------------------+
//| COMPLIANCE — Check Stop Policy                                   |
//+------------------------------------------------------------------+
bool ComplianceCheckStopPolicy(double newSL, string reason)
{
   if(!EnableComplianceAuditCounters) return true;
   g_complianceTotalChecks++;
   bool violation = false;
   if(EnforceNoSpreadSlStopMoves)
   {
      // Check if reason contains spread-triggered logic
      if(StringFind(reason, "SPREAD_TRIGGER") >= 0)
      {
         violation = true;
         g_ms.stopChangedDueSpread = true;
         LogDecisionEvent("COMPLIANCE", "VIOLATION: Stop move due to spread: " + reason, 0);
      }
   }
   if(EnforceNoSlippageStopMoves)
   {
      if(StringFind(reason, "SLIPPAGE_TRIGGER") >= 0)
      {
         violation = true;
         g_ms.stopChangedDueSlippage = true;
         LogDecisionEvent("COMPLIANCE", "VIOLATION: Stop move due to slippage: " + reason, 0);
      }
   }
   if(violation)
   {
      g_complianceTotalViolations++;
      g_ms.complianceViolationCount++;
      return false; // block the stop modification
   }
   return true;
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — MoveSL                                     |
//+------------------------------------------------------------------+
bool MoveSL(double newSL, string reason)
{
   if(!g_ms.active) return false;
   if(!PositionSelectByTicket(g_ms.ticket)) return false;
   // Compliance check
   if(!ComplianceCheckStopPolicy(newSL, reason))
   {
      if(EnableDebugPrints) Print("DEBUG: SL move BLOCKED by compliance: ", reason);
      return false;
   }
   double currentSL = PositionGetDouble(POSITION_SL);
   double currentTP = PositionGetDouble(POSITION_TP);
   newSL = NormalizePrice(newSL);
   // Don't move SL worse
   if(g_ms.direction == 1 && newSL <= currentSL && currentSL > 0) return false;
   if(g_ms.direction == -1 && newSL >= currentSL && currentSL > 0) return false;
   // Validate against stops level
   double ask = SymbolInfoDouble(InpSymbolName, SYMBOL_ASK);
   double bid = SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
   long stopsLevel = SymbolInfoInteger(InpSymbolName, SYMBOL_TRADE_STOPS_LEVEL);
   long freezeLevel = SymbolInfoInteger(InpSymbolName, SYMBOL_TRADE_FREEZE_LEVEL);
   double point = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
   double minDist = MathMax((double)stopsLevel, (double)freezeLevel) * point;
   if(g_ms.direction == 1)
   {
      if(bid - newSL < minDist) return false;
   }
   else
   {
      if(newSL - ask < minDist) return false;
   }
   if(g_trade.PositionModify(g_ms.ticket, newSL, currentTP))
   {
      LogDecisionEvent("MANAGEMENT", "SL moved to " + DoubleToString(newSL, (int)SymbolInfoInteger(InpSymbolName, SYMBOL_DIGITS)) + " reason=" + reason, g_ms.currentR);
      return true;
   }
   else
   {
      if(EnableDebugPrints) Print("DEBUG: SL modify failed, retcode=", g_trade.ResultRetcode());
      return false;
   }
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — PartialClose                               |
//+------------------------------------------------------------------+
bool PartialClosePercent(double pctOfOriginal, string reason)
{
   if(!EnablePartialClose || !g_ms.active) return false;
   if(!PositionSelectByTicket(g_ms.ticket)) return false;
   double currentVol = PositionGetDouble(POSITION_VOLUME);
   double closeVol = NormalizeLot(g_ms.initialVolume * pctOfOriginal / 100.0);
   if(closeVol > currentVol) closeVol = currentVol;
   bool result = g_trade.PositionClosePartial(g_ms.ticket, closeVol);
   if(!result)
   {
      LogDecisionEvent("MANAGEMENT", "Partial close failed ret=" + IntegerToString((int)g_trade.ResultRetcode()) + " " + RetcodeToString((int)g_trade.ResultRetcode()), g_ms.currentR);
      return false;
   }
   if(PositionSelectByTicket(g_ms.ticket))
   {
      double afterVol = PositionGetDouble(POSITION_VOLUME);
      if(afterVol > 0.0 && afterVol < g_ms.initialVolume) g_ms.initialVolume = MathMax(afterVol, SymbolInfoDouble(InpSymbolName, SYMBOL_VOLUME_MIN));
   }
   LogDecisionEvent("MANAGEMENT", "Partial close " + DoubleToString(closeVol,2) + " reason=" + reason, g_ms.currentR);
   return true;
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — ApplyBreakEven                             |
//+------------------------------------------------------------------+
void ApplyBreakEven()
{
   if(!EnableBreakEven) return;
   if(g_ms.beApplied) return;
   if(g_ms.barsInTrade < InpMinBarsBeforeBE) return;
   if(g_ms.currentR >= InpBETriggerR)
   {
      double bePrice;
      if(g_ms.direction == 1)
         bePrice = g_ms.entryPrice + InpBEBufferPoints * SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
      else
         bePrice = g_ms.entryPrice - InpBEBufferPoints * SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
      if(MoveSL(bePrice, "BREAK_EVEN"))
      {
         g_ms.beApplied = true;
         LogDecisionEvent("MANAGEMENT", "Break-even applied at " + DoubleToString(g_ms.currentR,2) + "R", g_ms.currentR);
      }
   }
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — ApplyPartialClose                          |
//+------------------------------------------------------------------+
void ApplyPartialClose()
{
   if(!EnablePartialClose) return;
   // Partial 1
   if(!g_ms.partial1Done && g_ms.currentR >= InpPartial1TriggerR)
   {
      if(PartialClosePercent(InpPartial1Pct, "TP1_HIT"))
         g_ms.partial1Done = true;
   }
   // Partial 2
   if(!g_ms.partial2Done && g_ms.currentR >= InpPartial2TriggerR)
   {
      if(PartialClosePercent(InpPartial2Pct, "TP2_HIT"))
         g_ms.partial2Done = true;
   }
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — ApplyTrailingLogic                         |
//+------------------------------------------------------------------+
void ApplyTrailingLogic(const MDAEFeatures &feat)
{
   double atr = 0;
   CopyBuf1(g_hATR, 0, 0, atr);
   if(atr <= 0) return;
   double ask = SymbolInfoDouble(InpSymbolName, SYMBOL_ASK);
   double bid = SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
   double currentPrice = (g_ms.direction == 1) ? bid : ask;
   ENUM_CLOSE_METHOD method = g_ms.activeMethod;
   // ATR Trail
   if(method == CM_ATR && EnableATRTrail)
   {
      double trailSL;
      if(g_ms.direction == 1)
         trailSL = currentPrice - atr * InpATRTrailMultiplier;
      else
         trailSL = currentPrice + atr * InpATRTrailMultiplier;
      MoveSL(trailSL, "ATR_TRAIL");
   }
   // Swing Trail
   if(method == CM_SWING && EnableSwingTrail)
   {
      int trailLookback = MathMax(2, InpTrailingStructureLookback);
      double swingLevel = 0;
      if(g_ms.direction == 1)
      {
         swingLevel = iLow(InpSymbolName, InpExecTF, 1);
         for(int i = 2; i <= trailLookback; i++)
         {
            double lo = iLow(InpSymbolName, InpExecTF, i);
            if(lo < swingLevel) swingLevel = lo;
         }
         double targetSL = swingLevel - atr * 0.2;
         LogDecisionEvent("MANAGEMENT", "SWING_TRAIL pivotLow=" + DoubleToString(swingLevel,_Digits) + " targetSL=" + DoubleToString(targetSL,_Digits), g_ms.currentR);
         MoveSL(targetSL, "SWING_TRAIL");
      }
      else
      {
         swingLevel = iHigh(InpSymbolName, InpExecTF, 1);
         for(int i = 2; i <= trailLookback; i++)
         {
            double hi = iHigh(InpSymbolName, InpExecTF, i);
            if(hi > swingLevel) swingLevel = hi;
         }
         double targetSL = swingLevel + atr * 0.2;
         LogDecisionEvent("MANAGEMENT", "SWING_TRAIL pivotHigh=" + DoubleToString(swingLevel,_Digits) + " targetSL=" + DoubleToString(targetSL,_Digits), g_ms.currentR);
         MoveSL(targetSL, "SWING_TRAIL");
      }
   }
   // Fast Bank
   if(method == CM_FAST && EnableFastBankMode)
   {
      double trailSL;
      double point = SymbolInfoDouble(InpSymbolName, SYMBOL_POINT);
      if(g_ms.direction == 1)
         trailSL = currentPrice - InpFastBankTrailPoints * point;
      else
         trailSL = currentPrice + InpFastBankTrailPoints * point;
      MoveSL(trailSL, "FAST_BANK_TRAIL");
   }
   // Hybrid: blend ATR and structure
   if(method == CM_HYBRID && EnableHybridCloseMode)
   {
      int trailLookback = MathMax(2, InpTrailingStructureLookback);
      double atrSL, structSL;
      if(g_ms.direction == 1)
      {
         atrSL = currentPrice - atr * InpATRTrailMultiplier;
         double recentLow = iLow(InpSymbolName, InpExecTF, 1);
         for(int i = 2; i <= trailLookback; i++)
         {
            double lo = iLow(InpSymbolName, InpExecTF, i);
            if(lo < recentLow) recentLow = lo;
         }
         structSL = recentLow - atr * 0.2;
         double bestSL = MathMax(atrSL, structSL);
         LogDecisionEvent("MANAGEMENT", "HYBRID_TRAIL pivotLow=" + DoubleToString(recentLow,_Digits) + " atrSL=" + DoubleToString(atrSL,_Digits) + " finalSL=" + DoubleToString(bestSL,_Digits), g_ms.currentR);
         MoveSL(bestSL, "HYBRID_TRAIL");
      }
      else
      {
         atrSL = currentPrice + atr * InpATRTrailMultiplier;
         double recentHigh = iHigh(InpSymbolName, InpExecTF, 1);
         for(int i = 2; i <= trailLookback; i++)
         {
            double hi = iHigh(InpSymbolName, InpExecTF, i);
            if(hi > recentHigh) recentHigh = hi;
         }
         structSL = recentHigh + atr * 0.2;
         double bestSL = MathMin(atrSL, structSL);
         LogDecisionEvent("MANAGEMENT", "HYBRID_TRAIL pivotHigh=" + DoubleToString(recentHigh,_Digits) + " atrSL=" + DoubleToString(atrSL,_Digits) + " finalSL=" + DoubleToString(bestSL,_Digits), g_ms.currentR);
         MoveSL(bestSL, "HYBRID_TRAIL");
      }
   }
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — CheckInvalidationExit                      |
//+------------------------------------------------------------------+
bool CheckInvalidationExit(const MDAEFeatures &feat)
{
   if(!EnableInvalidationExit) return false;
   // 1) Hard thesis break: regime flipped against position
   ENUM_REGIME currentRegime = DetectRegime();
   if(g_ms.direction == 1)
   {
      if(currentRegime == REGIME_MEAN_REVERSION || (currentRegime == REGIME_NO_TRADE && feat.continuationScore < InpContinuationThreshold))
      {
         LogDecisionEvent("MANAGEMENT", "REGIME_FLIP_INVALIDATION buy position currentRegime=" + EnumToString(currentRegime), g_ms.currentR);
         return true;
      }
   }
   else if(g_ms.direction == -1)
   {
      if(currentRegime == REGIME_TREND_EXPANSION && feat.continuationScore > InpContinuationThreshold)
      {
         LogDecisionEvent("MANAGEMENT", "REGIME_FLIP_INVALIDATION sell position currentRegime=" + EnumToString(currentRegime), g_ms.currentR);
         return true;
      }
   }
   // 2) Entropy/continuation breakdown
   if(feat.continuationScore < InpExhaustionThreshold && feat.entropyScore > InpEntropyHighThreshold)
   {
      LogDecisionEvent("MANAGEMENT", "Invalidation: High entropy + low continuation", g_ms.currentR);
      return true;
   }
   // 3) Sustained momentum collapse (speed near zero for extended period)
   if(feat.speedShort < feat.speedLong * 0.2 && feat.volumePulse < 0.5 && g_ms.currentR < 0.5)
   {
      LogDecisionEvent("MANAGEMENT", "Invalidation: Momentum collapse", g_ms.currentR);
      return true;
   }
   return false;
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — CheckTimeStopExit                          |
//+------------------------------------------------------------------+
bool CheckTimeStopExit()
{
   if(!EnableTimeStop) return false;
   if(g_ms.barsInTrade >= InpMaxTradeAgeBars)
   {
      // Only time-stop if trade isn't significantly profitable
      if(g_ms.currentR < 1.0)
      {
         LogDecisionEvent("MANAGEMENT", "Time stop: " + IntegerToString(g_ms.barsInTrade) + " bars, R=" + DoubleToString(g_ms.currentR,2), g_ms.currentR);
         return true;
      }
   }
   return false;
}
//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — Full Close                                 |
//+------------------------------------------------------------------+
void ClosePosition(string reason)
{
   if(!g_ms.active) return;
   bool result = g_trade.PositionClose(g_ms.ticket);
   if(result)
   {
      if(PositionSelectByTicket(g_ms.ticket))
      {
         g_ms.initialVolume = PositionGetDouble(POSITION_VOLUME);
      }
      else
      {
         g_ms.active = false;
      }
      LogDecisionEvent("EXIT", "Full close reason=" + reason, g_ms.currentR);
   }
   else
   {
      LogDecisionEvent("EXIT", "Full close failed ret=" + IntegerToString((int)g_trade.ResultRetcode()) + " " + RetcodeToString((int)g_trade.ResultRetcode()), g_ms.currentR);
   }
}
//+------------------------------------------------------------------+
//| MAIN POSITION MANAGEMENT ENGINE                                  |
//+------------------------------------------------------------------+
void ManagePosition()
{
   if(!g_ms.active) return;
   if(!PositionSelectByTicket(g_ms.ticket))
   {
      ulong relinked = FindManagedPositionTicket();
      if(relinked == 0 || !RefreshManagedStateFromPosition(relinked))
      {
         ResetManagedState();
         return;
      }
      LogDecisionEvent("RECONCILE", "Relinked managed ticket to active position " + IntegerToString((int)relinked), 0);
   }
   // Update live metrics
   double currentPrice;
   if(g_ms.direction == 1)
      currentPrice = SymbolInfoDouble(InpSymbolName, SYMBOL_BID);
   else
      currentPrice = SymbolInfoDouble(InpSymbolName, SYMBOL_ASK);
   double pnlPrice = (currentPrice - g_ms.entryPrice) * g_ms.direction;
   g_ms.currentR = (g_ms.riskDistance > 0) ? pnlPrice / g_ms.riskDistance : 0;
   if(pnlPrice > g_ms.mfe) g_ms.mfe = pnlPrice;
   if(pnlPrice < g_ms.mae) g_ms.mae = pnlPrice;
   // Update bars in trade
   datetime curBarTime = iTime(InpSymbolName, InpExecTF, 0);
   if(curBarTime > 0 && g_ms.entryTime > 0)
      g_ms.barsInTrade = (int)((curBarTime - g_ms.entryTime) / PeriodSeconds(InpExecTF));
   // Compute MDAE features
   MDAEFeatures feat;
   ComputeMDAEFeatures(feat);
   // Select close method with bounded switching
   if(EnableMDAE && EnablePolicySwitching)
   {
      ENUM_CLOSE_METHOD desired = SelectCloseMethod(feat);
      if(desired != g_ms.activeMethod)
      {
         if(g_ms.policySwitchCount < InpMaxPolicySwitches)
         {
            LogDecisionEvent("MDAE_METHOD", "Switch " + EnumToString(g_ms.activeMethod) + " -> " + EnumToString(desired) +
               " entropy=" + DoubleToString(feat.entropyScore,2) + " cont=" + DoubleToString(feat.continuationScore,2), g_ms.currentR);
            g_ms.activeMethod = desired;
            g_ms.policySwitchCount++;
         }
      }
   }
   // 1) Check invalidation exit
   if(CheckInvalidationExit(feat))
   {
      ClosePosition("INVALIDATION");
      return;
   }
   // 2) Check time stop
   if(CheckTimeStopExit())
   {
      ClosePosition("TIME_STOP");
      return;
   }
   // 3) If method is EXIT, close now
   if(g_ms.activeMethod == CM_EXIT)
   {
      ClosePosition("CM_EXIT");
      return;
   }
   // 4) Partials
   ApplyPartialClose();
   // 5) Break-even
   ApplyBreakEven();
   // 6) Trailing
   ApplyTrailingLogic(feat);
}
//+------------------------------------------------------------------+
//| LOGGING — LogDecisionEvent                                       |
//+------------------------------------------------------------------+
void LogDecisionEvent(string eventType, string detail, double score)
{
   if(!EnableLogging) return;
   string ts = TimeToString(TimeCurrent(), TIME_DATE|TIME_SECONDS);
   double spread = EstimateSpreadPoints();
   if(g_logHandle != INVALID_HANDLE)
   {
      FileWrite(g_logHandle, ts, InpSymbolName,
         (g_ms.active ? IntegerToString(g_ms.ticket) : "0"),
         eventType, EnumToString(g_currentRegime),
         (g_ms.active ? IntegerToString(g_ms.direction) : "0"),
         DoubleToString(g_ms.currentR, 3),
         EnumToString(g_ms.activeMethod),
         detail, "",
         DoubleToString(spread, 1), "",
         DoubleToString(score, 3),
         (g_ms.stopChangedDueSpread ? "true" : "false"),
         (g_ms.stopChangedDueSlippage ? "true" : "false"));
   }
   if(EnableDebugPrints)
   {
      Print("[", eventType, "] ", detail, " R=", DoubleToString(g_ms.currentR,2), " spread=", DoubleToString(spread,1));
   }
}
//+------------------------------------------------------------------+
//| LOGGING — LogTradeSummary                                        |
//+------------------------------------------------------------------+
void LogTradeSummary(string closeType, double profit)
{
   if(!EnableLogging) return;
   string ts = TimeToString(TimeCurrent(), TIME_DATE|TIME_SECONDS);
   string detail = "CloseType=" + closeType +
      " Profit=" + DoubleToString(profit, 2) +
      " MFE=" + DoubleToString(g_ms.mfe / g_ms.riskDistance, 2) + "R" +
      " MAE=" + DoubleToString(g_ms.mae / g_ms.riskDistance, 2) + "R" +
      " Bars=" + IntegerToString(g_ms.barsInTrade) +
      " Method=" + EnumToString(g_ms.activeMethod) +
      " Switches=" + IntegerToString(g_ms.policySwitchCount) +
      " ComplianceViolations=" + IntegerToString(g_ms.complianceViolationCount);
   if(g_logHandle != INVALID_HANDLE)
   {
      FileWrite(g_logHandle, ts, InpSymbolName, IntegerToString(g_ms.ticket),
         "EXIT_SUMMARY", EnumToString(g_currentRegime),
         IntegerToString(g_ms.direction),
         DoubleToString(g_ms.currentR, 3),
         EnumToString(g_ms.activeMethod),
         detail, closeType,
         DoubleToString(EstimateSpreadPoints(), 1), "",
         "0",
         (g_ms.stopChangedDueSpread ? "true" : "false"),
         (g_ms.stopChangedDueSlippage ? "true" : "false"));
   }
   Print("[TRADE_SUMMARY] ", detail);
}
