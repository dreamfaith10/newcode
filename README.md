# newcode
newcode
//+------------------------------------------------------------------+
//| TrendPullbackEA.mq5                                              |
//| Debug mode EA (trades every M15 bar, minimal filters)            |
//| Trades only the attached symbol (e.g., XAUUSD or EURUSD)         |
//| Fixed no-trade issue with simplified logic and enhanced logging   |
//+------------------------------------------------------------------+
#property copyright "Your Name"
#property link      "https://www.mql5.com"
#property version   "1.10"

//+------------------------------------------------------------------+
//| Input Parameters                                                 |
//+------------------------------------------------------------------+
input double InpLotSize          = 0.1;         // Lot size per trade (adjusted for XAUUSD)
input int    InpMaxOpenTrades    = 500;         // Max simultaneous trades
input double InpSL_ATR_Mult      = 10.0;        // SL = ATR × multiplier
input double InpRR_Ratio         = 3.0;         // Reward-to-risk ratio (TP = SL × 3)
input double InpMaxSpread        = 1000.0;      // Max spread (pips for FX, cents for XAU)
input bool   InpDebugMode        = true;        // Debug mode (bypass spread/margin filters)
input bool   InpShowDashboard    = true;        // Show on-chart dashboard

//+------------------------------------------------------------------+
//| Includes                                                         |
//+------------------------------------------------------------------+
#include <Trade\Trade.mqh>

//+------------------------------------------------------------------+
//| Global Variables                                                 |
//+------------------------------------------------------------------+
CTrade trade;
int handle_atr_m15;
double atr_m15[];
datetime last_bar_m15;
double point;
int digits;

//+------------------------------------------------------------------+
//| Expert initialization function                                    |
//+------------------------------------------------------------------+
int OnInit() {
   // Initialize symbol properties
   point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
   digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
   
   // Verify symbol and account
   if(SymbolInfoInteger(_Symbol, SYMBOL_TRADE_MODE) == SYMBOL_TRADE_MODE_DISABLED) {
      Print("Error: Symbol ", _Symbol, " is not tradable");
      return(INIT_FAILED);
   }
   if(!AccountInfoInteger(ACCOUNT_TRADE_ALLOWED)) {
      Print("Error: Trading not allowed for account");
      return(INIT_FAILED);
   }
   if(!AccountInfoInteger(ACCOUNT_TRADE_EXPERT)) {
      Print("Error: Expert Advisors not allowed for account");
      return(INIT_FAILED);
   }
   if(!SymbolInfoInteger(_Symbol, SYMBOL_TRADE_MODE) == SYMBOL_TRADE_MODE_FULL) {
      Print("Error: Symbol ", _Symbol, " market is closed");
      return(INIT_FAILED);
   }
   
   // Log symbol and account properties
   double min_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double max_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double step_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   double contract_size = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
   double margin_required = SymbolInfoDouble(_Symbol, SYMBOL_MARGIN_INITIAL);
   double stops_level = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * point;
   double current_spread = (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) / (digits == 5 ? 10.0 : 1.0);
   double free_margin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
   double account_balance = AccountInfoDouble(ACCOUNT_BALANCE);
   long account_limit_orders = AccountInfoInteger(ACCOUNT_LIMIT_ORDERS);
   Print(_Symbol, " Properties: Min Lot=", min_lot, ", Max Lot=", max_lot, ", Lot Step=", step_lot,
         ", Contract Size=", contract_size, ", Margin Required=", margin_required, 
         ", Stops Level=", stops_level, ", Current Spread=", current_spread, 
         ", Digits=", digits, ", Free Margin=", free_margin, 
         ", Balance=", account_balance, ", Account Max Orders=", account_limit_orders);
   
   // Initialize indicators
   handle_atr_m15 = iATR(_Symbol, PERIOD_M15, 14);
   if(handle_atr_m15 == INVALID_HANDLE) {
      Print("Failed to initialize ATR indicator for ", _Symbol);
      return(INIT_FAILED);
   }
   
   // Initialize trade object
   trade.SetExpertMagicNumber(123456);
   
   // Initialize last bar
   last_bar_m15 = 0;
   
   // Create dashboard if enabled
   if(InpShowDashboard) CreateDashboard();
   
   Print("EA initialized successfully for ", _Symbol, " on M15 chart");
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                  |
//+------------------------------------------------------------------+
void OnDeinit(const int reason) {
   // Clean up dashboard
   if(InpShowDashboard) ObjectsDeleteAll(0, "TrendPullbackEA_");
   Print("EA deinitialized for ", _Symbol, ", reason: ", reason);
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick() {
   // Check for new M15 bar
   datetime current_bar = iTime(_Symbol, PERIOD_M15, 0);
   if(current_bar == last_bar_m15) {
      Print(_Symbol, " No new M15 bar, skipping tick");
      return;
   }
   last_bar_m15 = current_bar;
   
   // Get indicator values
   if(!CopyBuffer(handle_atr_m15, 0, 0, 3, atr_m15)) {
      Print("Failed to copy ATR buffer for ", _Symbol);
      return;
   }
   
   // Log indicator values
   Print(_Symbol, " Indicators: ATR(M15)=", StringFormat("%.5f", atr_m15[1]));
   
   // Update dashboard
   if(InpShowDashboard) UpdateDashboard();
   
   // Apply filters
   int open_positions = CountOpenPositions();
   Print(_Symbol, " Filter Check: Open Trades=", open_positions, " (Max=", InpMaxOpenTrades, ")");
   if(open_positions >= InpMaxOpenTrades) {
      Print("Max open trades reached for ", _Symbol, ": ", open_positions);
      return;
   }
   
   if(!InpDebugMode) {
      double spread = (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) / (digits == 5 ? 10.0 : 1.0);
      Print(_Symbol, " Filter Check: Spread=", StringFormat("%.2f", spread), " (Max=", InpMaxSpread, ")");
      if(spread > InpMaxSpread) {
         Print("Spread too high for ", _Symbol, ": ", spread, " > ", InpMaxSpread);
         return;
      }
      
      double margin_required = SymbolInfoDouble(_Symbol, SYMBOL_MARGIN_INITIAL) * InpLotSize;
      double free_margin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
      Print(_Symbol, " Filter Check: Margin Required=", margin_required, ", Free Margin=", free_margin);
      if(margin_required > free_margin) {
         Print("Insufficient margin for ", _Symbol, ": Required=", margin_required, ", Available=", free_margin);
         return;
      }
   }
   
   // Entry logic (M15) - Trade every new bar, alternating long/short
   static bool trade_long = true;
   double price = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   bool long_signal = trade_long;
   bool short_signal = !trade_long;
   trade_long = !trade_long; // Alternate direction each bar
   Print(_Symbol, " Entry Check: Price=", StringFormat("%.5f", price), 
         ", Long Signal=", long_signal, ", Short Signal=", short_signal);
   
   if(long_signal) {
      OpenTrade(true, atr_m15[1]);
   }
   else if(short_signal) {
      OpenTrade(false, atr_m15[1]);
   }
}

//+------------------------------------------------------------------+
//| Count Open Positions for Current Symbol                          |
//+------------------------------------------------------------------+
int CountOpenPositions() {
   int count = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--) {
      ulong ticket = PositionGetTicket(i);
      if(PositionSelectByTicket(ticket) && PositionGetString(POSITION_SYMBOL) == _Symbol) {
         count++;
      }
   }
   return count;
}

//+------------------------------------------------------------------+
//| Open Trade                                                       |
//+------------------------------------------------------------------+
void OpenTrade(bool is_long, double atr) {
   Print(_Symbol, " Attempting to open ", is_long ? "Long" : "Short", " trade");
   
   double price = is_long ? SymbolInfoDouble(_Symbol, SYMBOL_ASK) : SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double sl = atr * InpSL_ATR_Mult;
   double tp = sl * InpRR_Ratio;
   double sl_price = is_long ? price - sl : price + sl;
   double tp_price = is_long ? price + tp : price - tp;
   
   // Normalize prices
   sl_price = NormalizeDouble(sl_price, digits);
   tp_price = NormalizeDouble(tp_price, digits);
   price = NormalizeDouble(price, digits);
   
   // Validate SL/TP against stops level
   double stops_level = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * point;
   double sl_distance = MathAbs(price - sl_price);
   double tp_distance = MathAbs(price - tp_price);
   Print(_Symbol, " Trade Validation: SL Distance=", sl_distance, ", TP Distance=", tp_distance, ", Stops Level=", stops_level);
   if(sl_distance < stops_level || tp_distance < stops_level) {
      Print("Invalid SL/TP for ", _Symbol, ": SL/TP too close to price, required distance=", stops_level);
      return;
   }
   
   Print(_Symbol, " Trade Parameters: Price=", price, ", SL=", sl_price, ", TP=", tp_price, ", Lot=", InpLotSize);
   
   // Validate lot size
   double min_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double max_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double step_lot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   if(InpLotSize < min_lot || InpLotSize > max_lot || fmod(InpLotSize, step_lot) != 0) {
      Print("Invalid lot size for ", _Symbol, ": ", InpLotSize, ", Min=", min_lot, ", Max=", max_lot, ", Step=", step_lot);
      return;
   }
   
   // Open position
   if(trade.PositionOpen(_Symbol, is_long ? ORDER_TYPE_BUY : ORDER_TYPE_SELL, InpLotSize, price, sl_price, tp_price, "TrendPullbackEA")) {
      Print(is_long ? "Long" : "Short", " trade opened for ", _Symbol, " at ", price, ", SL: ", sl_price, ", TP: ", tp_price);
   } else {
      int error = GetLastError();
      Print("Failed to open trade for ", _Symbol, ", error: ", error, " (", ErrorDescription(error), ")");
   }
}

//+------------------------------------------------------------------+
//| Dashboard Functions                                              |
//+------------------------------------------------------------------+
void CreateDashboard() {
   ObjectCreate(0, "TrendPullbackEA_Dashboard", OBJ_LABEL, 0, 0, 0);
   ObjectSetInteger(0, "TrendPullbackEA_Dashboard", OBJPROP_XDISTANCE, 10);
   ObjectSetInteger(0, "TrendPullbackEA_Dashboard", OBJPROP_YDISTANCE, 10);
   ObjectSetInteger(0, "TrendPullbackEA_Dashboard", OBJPROP_FONTSIZE, 10);
   ObjectSetString(0, "TrendPullbackEA_Dashboard", OBJPROP_FONT, "Arial");
}

void UpdateDashboard() {
   string text = "TrendPullbackEA Dashboard\n";
   text += "Symbol: " + _Symbol + "\n";
   text += StringFormat("Spread: %.2f (Max: %.2f)\n", (double)SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) / (digits == 5 ? 10.0 : 1.0), InpMaxSpread);
   text += "Open Trades: " + IntegerToString(CountOpenPositions()) + " (Max: " + IntegerToString(InpMaxOpenTrades) + ")\n";
   text += StringFormat("ATR (M15): %.2f\n", atr_m15[0] / point);
   text += "Debug Mode: " + (InpDebugMode ? "Enabled" : "Disabled") + "\n";
   ObjectSetString(0, "TrendPullbackEA_Dashboard", OBJPROP_TEXT, text);
}

//+------------------------------------------------------------------+
//| Error Description Function                                       |
//+------------------------------------------------------------------+
string ErrorDescription(int error_code) {
   switch(error_code) {
      case 10004: return "Requote";
      case 10006: return "Request rejected";
      case 10007: return "Request canceled";
      case 10009: return "Request accepted";
      case 10013: return "Invalid request parameters";
      case 10014: return "Invalid volume";
      case 10015: return "Invalid price";
      case 10016: return "Invalid stops";
      case 10018: return "Market closed";
      case 10019: return "No money";
      case 10021: return "Price changed";
      case 10030: return "Invalid expiration";
      default: return "Unknown error";
   }
}
