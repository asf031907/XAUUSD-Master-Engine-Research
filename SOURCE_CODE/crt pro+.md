//@version=6
indicator("CRT Pro+ by Artas", shorttitle = "CRT Pro+", overlay = true, max_bars_back = 5000, max_boxes_count = 500, max_lines_count = 500, max_labels_count = 500)

type Candle
    float           o
    float           c
    float           h
    float           l
    int             o_idx
    box             body
    line            wick_up
    line            wick_down
    label           sweepLbl

type CandleSettings
    bool            show
    string          htf
    int             max_display

type Settings
    color           bull_body
    color           bull_border
    color           bull_wick
    color           bear_body
    color           bear_border
    color           bear_wick
    int             offset
    int             buffer
    int             width

type CandleSet
    Candle[]        candles
    CandleSettings  settings
    label           tfLabel
    line[]          sweepLines

type SMT
    string smtType
    string smtPair
    float prevChartHigh
    float currChartHigh
    int prevChartHighTime
    int currChartHighTime
    float prevChartLow
    float currChartLow
    int prevChartLowTime
    int currChartLowTime
    bool confirmed
    int creationBar
    line smtLine
    label smtLabel

type CRTModel
    box             rangeBox
    line            separatorLine
    line            sweepLine
    label           sweepLabel
    line            targetLine
    label           targetLabel
    line            cisdLine
    label           cisdLabel
    label           c1Label
    label           c2Label
    label           c3Label
    string          bias         = "Neutral"
    string          status       = "Formation"
    float           h            = na
    float           l            = na
    float           o            = na
    float           c            = na
    int             startBar     = na
    int             endBar       = na
    int             h_bar        = na
    int             l_bar        = na
    float           h_bar_open   = na
    float           l_bar_open   = na
    float           manipOpen    = na
    int             manipBar     = na
    bool            sweptHigh    = false
    bool            sweptLow     = false
    bool            dpurge       = false
    bool            cisdDone     = false
    int             cisdBar      = na
    float           cisdLevel    = na

G_TOP = "Global Model Settings"
modelMode        = input.string("Model 1", "Model Mode Selector", options = ["Model 1", "Model 2"], group = G_TOP)
biasFilter       = input.string("Neutral", "Bias Filter Mode", options = ["Neutral", "Bullish", "Bearish"], group = G_TOP)

G_TF = "Timeframe & Pairing"
tf_preset        = input.string("Auto", "Timeframe Pairing", options = ["Auto", "1H - 1D", "4H - 1W", "1D - 1M", "1W - 3M", "Custom"], group = G_TF)
custom_htf       = input.timeframe("1W", "Custom HTF (If Custom Mode)", group = G_TF)
max_display_inp  = input.int(4, "Max Displayed HTF Candles (Right Margin)", minval = 1, maxval = 50, group = G_TF)
history_lookback = input.int(2, "Model History Lookback", minval = 1, maxval = 200, group = G_TF)

G_APP = "HTF Candle Appearance"
bull_body_col   = input.color(color.new(#70a880, 0), "Body Bull", inline = "body", group = G_APP)
bear_body_col   = input.color(color.new(#000000, 0), "Body Bear", inline = "body", group = G_APP)
bull_bord_col   = input.color(color.new(#000000, 0), "Borders Bull", inline = "bord", group = G_APP)
bear_bord_col   = input.color(color.new(#000000, 0), "Borders Bear", inline = "bord", group = G_APP)
bull_wick_col   = input.color(color.new(#000000, 0), "Wick Bull", inline = "wick", group = G_APP)
bear_wick_col   = input.color(color.new(#000000, 0), "Wick Bear", inline = "wick", group = G_APP)
chart_offset    = input.int(15, "Chart Padding (Offset)", minval = 1, group = G_APP)
candle_buffer   = input.int(1, "Candle Spacing (Buffer)", minval = 1, maxval = 4, group = G_APP)
candle_width    = input.int(1, "Candle Width (Multiplier)", minval = 1, maxval = 4, group = G_APP) * 2

G_CRT = "CRT Box & Separators"
show_crt_box     = input.bool(false, "Show CRT Boxes (Overrides Separators)", group = G_CRT)
sep_col          = input.color(color.new(#787b86, 50), "Separator Color", group = G_CRT)
sep_width        = input.int(1, "Separator Width", minval = 1, maxval = 5, group = G_CRT)
color_by_bias    = input.bool(false, "Color Boxes By Bias", group = G_CRT)
neutral_box_col  = input.color(color.new(#787b86, 93), "Neutral Box Color", group = G_CRT)
crt_bull_col     = input.color(color.new(#2962ff, 93), "Bullish Box (If Colored By Bias)", group = G_CRT)
crt_bear_col     = input.color(color.new(#f77c80, 91), "Bearish Box (If Colored By Bias)", group = G_CRT)
crt_border_col   = input.color(color.new(#787b86, 100), "Box Border", group = G_CRT)
show_crt_labels  = input.bool(true, "Show CRT H / L Labels", group = G_CRT)
label_text_col   = input.color(color.new(#000000, 10), "CRT Label Text Color", group = G_CRT)
label_size_inp   = input.string("Small", "Label Size", options = ["Tiny", "Small", "Normal", "Large"], group = G_CRT)
hide_overlap     = input.bool(false, "Hide Overlapped Boxes", group = G_CRT)

G_SWEEP = "Sweeps"
show_sweep_lines = input.bool(true, "Show Sweep Lines", group = G_SWEEP)
sweep_bull_col   = input.color(color.new(#000000, 0), "Bullish Sweep Color", group = G_SWEEP)
sweep_bear_col   = input.color(color.new(#000000, 0), "Bearish Sweep Color", group = G_SWEEP)
show_dpurge      = input.bool(true, "Show D-Purge Labels", group = G_SWEEP)
dpurge_col       = input.color(color.new(#000000, 0), "D-Purge Color", group = G_SWEEP)

G_CISD = "CISD"
show_cisd            = input.bool(true, "Show CISD Line", group = G_CISD)
cisd_bull_col        = input.color(#3b3eff, "Bullish CISD", group = G_CISD)
cisd_bear_col        = input.color(color.new(#f23645, 0), "Bearish CISD", group = G_CISD)
cisd_length          = input.int(15, "CISD Line Length (bars)", minval = 5, maxval = 100, group = G_CISD)
cisd_series_lookback = input.int(50, "CISD Series Max Lookback (bars)", minval = 5, maxval = 300, tooltip = "How far back the script searches for the consecutive same-direction candle series that preceded the sweep.", group = G_CISD)

G_NEW_OB = "MODEL 1 - ORDER BLOCK SETTINGS (AFTER SWEEP ONLY)"
showBullOB       = input.bool(true, "Show Bullish OB on Sweep", group = G_NEW_OB)
showBearOB       = input.bool(true, "Show Bearish OB on Sweep", group = G_NEW_OB)
bullOBColor      = input.color(color.new(#3b3eff, 0), "Bullish OB Line Color", group = G_NEW_OB)
bearOBColor      = input.color(color.new(#f23645, 0), "Bearish OB Line Color", group = G_NEW_OB)
maxObCount       = input.int(1, "Max Displayed OB Lines", minval = 1, maxval = 100, group = G_NEW_OB)
obBarsToExtend   = input.int(8, "OB Line Extend (Bars)", minval = 1, maxval = 100, group = G_NEW_OB)
obLineWidth      = input.int(1, "OB Line Width", minval = 1, maxval = 10, group = G_NEW_OB)
obTextSize       = input.string("Small", "OB Text Size", options = ["Tiny", "Small", "Normal", "Large"], group = G_NEW_OB)

G_MODEL = "Model Labels & Status Colors"
show_model_labels = input.bool(true, "Show C1 / C2 / C3 Labels", group = G_MODEL)
grey_col          = input.color(color.new(#787b86, 0), "Formation / Active Color", group = G_MODEL)
green_col         = input.color(color.new(#089981, 0), "Success Color", group = G_MODEL)
red_col           = input.color(color.new(#f23645, 0), "Invalidated Color", group = G_MODEL)
sync_right_panel  = input.bool(true, "Sync Right-Margin Candle Borders With Model Status", group = G_MODEL)

G_INFO = "HTF Info Panel (Above Candles)"
show_big_tf_label = input.bool(true, "Show Big Timeframe Label", group = G_INFO)
big_tf_size_i     = input.string("Large", "Big Label Size", options = ["Normal", "Large", "Huge"], group = G_INFO)
big_tf_color      = input.color(color.new(#131722, 0), "Big Label Color", group = G_INFO)
show_info_text    = input.bool(true, "Show Pairing / Countdown / Bias Text", group = G_INFO)
info_text_size_i  = input.string("Small", "Info Text Size", options = ["Tiny", "Small", "Normal"], group = G_INFO)
info_text_color   = input.color(color.new(#131722, 0), "Info Text Color", group = G_INFO)
show_panel_sweep  = input.bool(true, "Show 'Sweep' Label On HTF Candles", group = G_INFO)
panel_sweep_size  = input.string("Tiny", "Sweep Label Size", options = ["Tiny", "Small"], group = G_INFO)

G_SMT = "SMT Settings"
enableSMT         = input.bool(true, 'Enable SMT Detection', group = G_SMT)
smtAutoMode       = input.bool(true, 'SMT Auto Pair', group = G_SMT)
smtManualPair     = input.symbol('', 'SMT Manual Pair', group = G_SMT)
smtLineWidth      = input.int(1, 'SMT Line Width', minval = 1, group = G_SMT)
smtKeepOnlyLatest = input.bool(true, 'Keep Only Latest SMTs', group = G_SMT)

G_DASH = "Dashboard"
show_dashboard  = input.bool(true, "Show Dashboard", group = G_DASH)
dash_position_i = input.string("Top Right", "Position", options = ["Top Right", "Top Center", "Top Left", "Bottom Right", "Bottom Center", "Bottom Left"], group = G_DASH)
dash_size_i     = input.string("Small", "Text Size", options = ["Tiny", "Small", "Normal"], group = G_DASH)

f_lineStyle(string s) =>
    s == "Solid" ? line.style_solid : s == "Dashed" ? line.style_dashed : line.style_dotted

f_labelSize(string s) =>
    s == "Tiny" ? size.tiny : s == "Small" ? size.small : s == "Normal" ? size.normal : size.large

f_dashSize(string s) =>
    s == "Tiny" ? size.tiny : s == "Small" ? size.small : size.normal

f_dashPos(string s) =>
    s == "Top Right" ? position.top_right : s == "Top Center" ? position.top_center : s == "Top Left" ? position.top_left : s == "Bottom Right" ? position.bottom_right : s == "Bottom Center" ? position.bottom_center : position.bottom_left

f_friendlyTF(string tf) =>
    string res = tf
    if tf == "D"
        res := "1D"
    else if tf == "W"
        res := "1W"
    else if tf == "M"
        res := "1M"
    else
        float val = str.tonumber(tf)
        if not na(val)
            if val >= 1440 and val % 1440 == 0
                res := str.tostring(int(val / 1440)) + "D"
            else if val >= 60 and val % 60 == 0
                res := str.tostring(int(val / 60)) + "H"
            else
                res := str.tostring(int(val)) + "m"
    res

f_countdownClean(string tf) =>
    string txt = "N/A"
    t_close = time_close(tf)
    if not na(t_close)
        timeRemaining = (t_close - timenow) / 1000
        if timeRemaining > 0
            int days    = int(math.floor(timeRemaining / 86400))
            int hours   = int(math.floor((timeRemaining - (days * 86400)) / 3600))
            int minutes = int(math.floor((timeRemaining - (days * 86400) - (hours * 3600)) / 60))
            int seconds = int(math.floor(timeRemaining - (days * 86400) - (hours * 3600) - (minutes * 60)))
            
            r = str.tostring(seconds, "00")
            if minutes > 0 or hours > 0 or days > 0
                r := str.tostring(minutes, "00") + ":" + r
            if hours > 0 or days > 0
                r := str.tostring(hours, "00") + ":" + r
            if days > 0
                r := str.tostring(days) + "D " + r
            txt := r
    txt

getSMTPair() =>
    string ticker = syminfo.ticker
    string res = ""
    if str.contains(ticker, "MNQ1!")
        res := "CME_MINI:MES1!"
    else if str.contains(ticker, "MES1!")
        res := "CME_MINI:MNQ1!"
    else if str.contains(ticker, "MYM1!")
        res := "CME_MINI:MNQ1!"
    else if str.contains(ticker, "NQ1!") or str.contains(ticker, "NDX") or str.contains(ticker, "QQQ") or ticker == "NQ"
        res := "CME_MINI:ES1!"
    else if str.contains(ticker, "ES1!") or str.contains(ticker, "SPX") or str.contains(ticker, "SPY") or ticker == "ES"
        res := "CME_MINI:NQ1!"
    else if str.contains(ticker, "YM1!")
        res := "CME_MINI:NQ1!"
    else if str.contains(ticker, "RTY1!")
        res := "CBOT:YM1!"
    else if str.contains(ticker, "NAS100")
        res := "SPX500"
    else if str.contains(ticker, "SPX500")
        res := "NAS100"
    else if str.contains(ticker, "US30")
        res := "NAS100"
    else if str.contains(ticker, "EURUSD")
        res := "GBPUSD"
    else if str.contains(ticker, "GBPUSD")
        res := "EURUSD"
    else if str.contains(ticker, "EURCHF")
        res := "GBPCHF"
    else if str.contains(ticker, "GBPCHF")
        res := "EURCHF"
    else if str.contains(ticker, "EURCAD")
        res := "GBPCAD"
    else if str.contains(ticker, "GBPCAD")
        res := "EURCAD"
    else if str.contains(ticker, "AUDUSD")
        res := "NZDUSD"
    else if str.contains(ticker, "NZDUSD")
        res := "AUDUSD"
    else if str.contains(ticker, "EURAUD")
        res := "GBPAUD"
    else if str.contains(ticker, "GBPAUD")
        res := "EURAUD"
    else if str.contains(ticker, "AUDCAD")
        res := "NZDCAD"
    else if str.contains(ticker, "NZDCAD")
        res := "AUDCAD"
    else if str.contains(ticker, "EURJPY")
        res := "GBPJPY"
    else if str.contains(ticker, "GBPJPY")
        res := "EURJPY"
    else if str.contains(ticker, "AUDJPY")
        res := "NZDJPY"
    else if str.contains(ticker, "NZDJPY")
        res := "AUDJPY"
    else if str.contains(ticker, "EURNZD")
        res := "GBPNZD"
    else if str.contains(ticker, "GBPNZD")
        res := "EURNZD"
    else if str.contains(ticker, "AUDCHF")
        res := "NZDCHF"
    else if str.contains(ticker, "NZDCHF")
        res := "AUDCHF"
    else if str.contains(ticker, "CADJPY")
        res := "CHFJPY"
    else if str.contains(ticker, "CHFJPY")
        res := "CADJPY"
    else if str.contains(ticker, "USDJPY")
        res := "EURJPY"
    else if str.contains(ticker, "USDCAD")
        res := "EURCAD"
    else if str.contains(ticker, "USDCHF")
        res := "EURCHF"
    else if str.contains(ticker, "6E1!") or ticker == "6E"
        res := "CME:6B1!"
    else if str.contains(ticker, "6B1!") or ticker == "6B"
        res := "CME:6E1!"
    else if str.contains(ticker, "6N1!") or ticker == "6N"
        res := "CME:6A1!"
    else if str.contains(ticker, "6A1!") or ticker == "6A"
        res := "CME:6N1!"
    else if str.contains(ticker, "XAUUSD")
        res := "XAGUSD"
    else if str.contains(ticker, "XAGUSD")
        res := "XAUUSD"
    else if str.contains(ticker, "XAUEUR")
        res := "XAUUSD"
    else if str.contains(ticker, "XAUGBP")
        res := "COMEX:GC1!"
    else if str.contains(ticker, "GC1!")
        res := "XAGUSD"
    else if str.contains(ticker, "XAGEUR")
        res := "COMEX:SI1!"
    else if str.contains(ticker, "XAGGBP")
        res := "COMEX:SI1!"
    else if str.contains(ticker, "SIL1!") or str.contains(ticker, "SI1!")
        res := "XAGEUR"
    else if str.contains(ticker, "CL1!")
        res := "NYMEX:RB1!"
    else if str.contains(ticker, "RB1!")
        res := "NYMEX:HO1!"
    else if str.contains(ticker, "HO1!")
        res := "NYMEX:CL1!"
    else if str.contains(ticker, "MBT1!")
        res := "CME:MET1!"
    else if str.contains(ticker, "MET1!")
        res := "CME:MBT1!"
    else if str.contains(ticker, "BTCUSDT") or str.contains(ticker, "BTC")
        res := "ETHUSDT"
    else if str.contains(ticker, "ETHUSDT") or str.contains(ticker, "ETH")
        res := "BTCUSDT"
    else
        res := "CME_MINI:ES1!"
    res

f_candlesHigh(Candle[] candles) =>
    float h = na
    if array.size(candles) > 0
        for i = 0 to array.size(candles) - 1
            Candle c = candles.get(i)
            h := na(h) ? c.h : math.max(h, c.h)
    h

f_newModel(float o0, float h0, float l0, float c0, int startBar0, color boxCol, color borderCol, bool showBox, color sepCol, int sepW) =>
    CRTModel m = CRTModel.new(o = o0, h = h0, l = l0, c = c0, startBar = startBar0, endBar = startBar0, h_bar = startBar0, l_bar = startBar0, h_bar_open = o0, l_bar_open = o0)
    color finalBgCol = (showBox or color_by_bias) ? boxCol : color.new(color.white, 100)
    color finalBordCol = (showBox or color_by_bias) ? borderCol : color.new(color.white, 100)
    m.rangeBox := box.new(left = startBar0, top = h0, right = startBar0, bottom = l0, border_color = finalBordCol, border_width = 1, bgcolor = finalBgCol)
    
    if not (showBox or color_by_bias)
        m.separatorLine := line.new(x1 = startBar0, y1 = l0, x2 = startBar0, y2 = h0, color = sepCol, width = sepW, extend = extend.both, style = line.style_solid)
    m

f_deleteModel(CRTModel m) =>
    box.delete(m.rangeBox)
    line.delete(m.separatorLine)
    line.delete(m.sweepLine)
    label.delete(m.sweepLabel)
    line.delete(m.targetLine)
    label.delete(m.targetLabel)
    line.delete(m.cisdLine)
    label.delete(m.cisdLabel)
    label.delete(m.c1Label)
    label.delete(m.c2Label)
    label.delete(m.c3Label)

isValidTimeframe(string HTF) =>
    n1 = timeframe.in_seconds()
    n2 = timeframe.in_seconds(HTF)
    (n2 >= timeframe.in_seconds("1D") and n2 > n1) or (n1 < n2 and n2 > 0 and math.round(n2 / n1) == n2 / n1)

f_cisdLevel(int extremeBar, bool wasHighSweep) =>
    int off0 = bar_index - extremeBar
    float body_hi = math.max(open[off0], close[off0])
    float body_lo = math.min(open[off0], close[off0])
    int origin_bar = extremeBar
    
    if off0 >= 0 and off0 <= bar_index
        for i = 1 to cisd_series_lookback
            int off = off0 + i
            if off > bar_index
                break
            bool bull_i = close[off] > open[off]
            bool keepGoing = wasHighSweep ? bull_i : not bull_i
            if keepGoing
                body_hi := math.max(body_hi, math.max(open[off], close[off]))
                body_lo := math.min(body_lo, math.min(open[off], close[off]))
                origin_bar := bar_index - off
            else
                break
    [wasHighSweep ? body_lo : body_hi, origin_bar]

var Settings settings = Settings.new(
     bull_body_col, bull_bord_col, bull_wick_col,
     bear_body_col, bear_bord_col, bear_wick_col,
     chart_offset, candle_buffer, candle_width)

f_syncCandlePanel(CandleSet candleSet, int startBarRef, color borderCol, bool isSweep, bool isHigh, color sweepCol, bool showLbl, string lblSize) =>
    if array.size(candleSet.candles) > 0
        for i = 0 to array.size(candleSet.candles) - 1
            Candle cnd = candleSet.candles.get(i)
            if cnd.o_idx == startBarRef
                box.set_border_color(cnd.body, borderCol)
                if not na(cnd.sweepLbl)
                    label.delete(cnd.sweepLbl)
                    cnd.sweepLbl := na
                if isSweep and showLbl
                    float lblY = isHigh ? cnd.h : cnd.l
                    string lblStyle = isHigh ? label.style_label_down : label.style_label_up
                    int midX = int(box.get_left(cnd.body) + settings.width / 2)
                    cnd.sweepLbl := label.new(x = midX, y = lblY, text = "Sweep", style = lblStyle, color = color.new(color.white, 100), textcolor = sweepCol, size = lblSize)

tf_secs = timeframe.in_seconds(timeframe.period)
auto_tf = tf_secs <= timeframe.in_seconds("1")    ? "15"  :
          tf_secs <= timeframe.in_seconds("3")    ? "30"  :
          tf_secs <= timeframe.in_seconds("5")    ? "60"  :
          tf_secs <= timeframe.in_seconds("15")   ? "240" :
          tf_secs <= timeframe.in_seconds("60")   ? "1D"  :
          tf_secs <= timeframe.in_seconds("480")  ? "1W"  :
          tf_secs <= timeframe.in_seconds("1440") ? "1M"  : "1W"

preset_htf = tf_preset == "1H - 1D" ? "1D" :
             tf_preset == "4H - 1W" ? "1W" :
             tf_preset == "1D - 1M" ? "1M" :
             tf_preset == "1W - 3M" ? "3M" :
             tf_preset == "Custom"  ? custom_htf : na

string htf_str = tf_preset == "Auto" ? auto_tf : preset_htf

var bool rtHighBreached = false
var bool rtLowBreached  = false
var bool rtSweptHigh    = false
var bool rtSweptLow     = false
var line rtObLine       = na
var label rtObLabel     = na
var float rtSweepHighLevel = na
var float rtSweepLowLevel  = na

var line[] obLinesArray   = array.new<line>(0)
var label[] obLabelsArray = array.new<label>(0)

f_drawRealtimeOB(bool isBull, float level, int startBar, color col, int width, int extension, string txtSize, color txtColor) =>
    string arrowSymbol = isBull ? '🡅' : '🡇'
    string labelText = "OB " + arrowSymbol
    int endBar = bar_index + extension
    
    line targetLine = line.new(startBar, level, endBar, level, xloc=xloc.bar_index, extend=extend.none, color=col, width=width)
    label targetLabel = label.new(
         x = endBar, 
         y = level, 
         text = labelText, 
         xloc = xloc.bar_index, 
         yloc = yloc.price, 
         color = color.new(color.white, 100), 
         textcolor = txtColor,
         style = label.style_label_left, 
         textalign = text.align_left,
         size = f_labelSize(txtSize))
    [targetLine, targetLabel]

var CandleSet htf1 = CandleSet.new(array.new<Candle>(0), CandleSettings.new(true, htf_str, max_display_inp), na, array.new<line>(0))
htf1.settings.htf := htf_str

var CRTModel curModel  = CRTModel.new()
var CRTModel prevModel = CRTModel.new()
var CRTModel[] modelHistory = array.new<CRTModel>(0)

var string globalBias = "Neutral"

var bool alertFormed        = false
var bool alertSuccess       = false
var bool alertInvalidated   = false
var bool alertSweepEvent    = false
var bool alertDpurgeEvent   = false

alertFormed        := false
alertSuccess       := false
alertInvalidated   := false
alertSweepEvent    := false
alertDpurgeEvent   := false

var label bigTfLabel     = na
var label biasTextLabel  = na

var float chartPivotHigh = na, var float chartPivotLow = na
var int chartPivotHighTime = na, var int chartPivotLowTime = na
var float smtPivotHigh = na, var float smtPivotLow = na
var float prevChartPivotHigh = na, var float prevChartPivotLow = na
var int prevChartPivotHighTime = na, var int prevChartPivotLowTime = na
var float prevSmtPivotHigh = na, var float prevSmtPivotLow = na

var smtArray = array.new<SMT>()
var smtLabels = array.new<label>()
var float prevCH_smt = na, var int prevCHT_smt = na
var float currCH_smt = na, var int currCHT_smt = na
var float prevCL_smt = na, var int prevCLT_smt = na
var float currCL_smt = na, var int currCLT_smt = na
var line liveBearSmtL = na
var line liveBullSmtL = na

method Reorder(CandleSet candleSet, int offset) =>
    size = candleSet.candles.size()
    if size > 0
        for i = size - 1 to 0
            Candle candle = candleSet.candles.get(i)
            t_buffer = offset + ((settings.width + settings.buffer) * (size - i - 1))
            posX = bar_index + t_buffer
            midX = posX + (settings.width / 2)

            box.set_left(candle.body, posX)
            box.set_right(candle.body, posX + settings.width)
            line.set_x1(candle.wick_up, midX)
            line.set_x2(candle.wick_up, midX)
            line.set_x1(candle.wick_down, midX)
            line.set_x2(candle.wick_down, midX)
            if not na(candle.sweepLbl)
                label.set_x(candle.sweepLbl, int(midX))

    if candleSet.sweepLines.size() > 0
        for ln in candleSet.sweepLines
            line.delete(ln)
        candleSet.sweepLines.clear()

    if size > 1
        for i = 0 to size - 2
            Candle curr = candleSet.candles.get(i)
            Candle prev = candleSet.candles.get(i + 1)
            
            curr_left  = box.get_left(curr.body)
            prev_left  = box.get_left(prev.body)
            curr_right = box.get_right(curr.body)

            if curr.h > prev.h and curr.c < prev.h
                ln = line.new(prev_left + settings.width / 2, prev.h, curr_right, prev.h, color = color.new(color.black, 30), style = line.style_solid, width = 1)
                candleSet.sweepLines.push(ln)
            if curr.l < prev.l and curr.c > prev.l
                ln = line.new(prev_left + settings.width / 2, prev.l, curr_right, prev.l, color = color.new(color.black, 30), style = line.style_solid, width = 1)
                candleSet.sweepLines.push(ln)

method Monitor(CandleSet candleSet) =>
    isNew = ta.change(time(candleSet.settings.htf)) != 0
    if isNew
        Candle candle = Candle.new(open, close, high, low, bar_index)
        bull = candle.c > candle.o
        candle.body := box.new(bar_index, math.max(candle.o, candle.c), bar_index + 2, math.min(candle.o, candle.c),
             bull ? settings.bull_border : settings.bear_border, 1, bgcolor = bull ? settings.bull_body : settings.bear_body)
        candle.wick_up := line.new(bar_index, candle.h, bar_index, math.max(candle.o, candle.c), color = bull ? settings.bull_wick : settings.bear_wick)
        candle.wick_down := line.new(bar_index, math.min(candle.o, candle.c), bar_index, candle.l, color = bull ? settings.bull_wick : settings.bear_wick)
        candleSet.candles.unshift(candle)
        if candleSet.candles.size() > candleSet.settings.max_display
            Candle del = candleSet.candles.pop()
            box.delete(del.body)
            line.delete(del.wick_up)
            line.delete(del.wick_down)
            label.delete(del.sweepLbl)
    candleSet

method Update(CandleSet candleSet, int offset) =>
    if candleSet.candles.size() > 0
        Candle candle = candleSet.candles.get(0)
        candle.h := math.max(high, candle.h)
        candle.l := math.min(low, candle.l)
        candle.c := close
        bull = candle.c > candle.o
        box.set_top(candle.body, math.max(candle.o, candle.c))
        box.set_bottom(candle.body, math.min(candle.o, candle.c))
        box.set_bgcolor(candle.body, bull ? settings.bull_body : settings.bear_body)
        line.set_y1(candle.wick_up, candle.h)
        line.set_y2(candle.wick_up, math.max(candle.o, candle.c))
        line.set_y1(candle.wick_down, candle.l)
        line.set_y2(candle.wick_down, math.min(candle.o, candle.c))
        if barstate.islast
            candleSet.Reorder(offset)
    candleSet

bool validTF = isValidTimeframe(htf_str)

if validTF
    bool isNewHTF = ta.change(time(htf_str)) != 0
    string labelSz = f_labelSize(label_size_inp)

    if isNewHTF
        line.delete(rtObLine)
        label.delete(rtObLabel)
        line.delete(curModel.sweepLine)
        line.delete(curModel.targetLine)
        label.delete(curModel.sweepLabel)
        label.delete(curModel.targetLabel)

        rtHighBreached := false
        rtLowBreached  := false
        rtSweptHigh    := false
        rtSweptLow     := false
        rtObLine       := na
        rtObLabel      := na
        rtSweepHighLevel := na
        rtSweepLowLevel  := na

        if not na(curModel.rangeBox)
            curModel.endBar := bar_index - 1
            box.set_right(curModel.rangeBox, curModel.endBar)

            if not na(prevModel.rangeBox)
                curModel.sweptHigh := (curModel.h > prevModel.h and curModel.c < prevModel.h) and (biasFilter == "Neutral" or biasFilter == "Bearish")
                curModel.sweptLow  := (curModel.l < prevModel.l and curModel.c > prevModel.l) and (biasFilter == "Neutral" or biasFilter == "Bullish")
                curModel.dpurge    := curModel.sweptHigh and curModel.sweptLow

                if curModel.dpurge
                    curModel.bias := curModel.c > curModel.o ? "Bullish" : "Bearish"
                else if curModel.sweptHigh
                    curModel.bias := "Bearish"
                else if curModel.sweptLow
                    curModel.bias := "Bullish"
                else
                    curModel.bias := "Neutral"

                curModel.status := (curModel.sweptHigh or curModel.sweptLow) ? "Active" : "Formation"
                globalBias := curModel.bias

                color boxFillCol = color_by_bias ? (curModel.bias == "Bullish" ? crt_bull_col : curModel.bias == "Bearish" ? crt_bear_col : neutral_box_col) : neutral_box_col
                box.set_bgcolor(curModel.rangeBox, (show_crt_box or color_by_bias) ? boxFillCol : color.new(color.white, 100))

                color initBorderCol = (curModel.sweptHigh or curModel.sweptLow) ? grey_col : crt_border_col
                box.set_border_color(curModel.rangeBox, (show_crt_box or color_by_bias) ? initBorderCol : color.new(color.white, 100))

                if curModel.sweptHigh or curModel.sweptLow
                    int extremeBarRef = curModel.sweptHigh ? curModel.h_bar : curModel.l_bar
                    [lvl, oBar] = f_cisdLevel(extremeBarRef, curModel.sweptHigh)
                    curModel.manipOpen := lvl
                    curModel.manipBar  := oBar

                color sweepMarkCol = curModel.sweptHigh ? sweep_bear_col : sweep_bull_col
                if sync_right_panel
                    f_syncCandlePanel(htf1, curModel.startBar, initBorderCol, (curModel.sweptHigh or curModel.sweptLow), curModel.sweptHigh, sweepMarkCol, show_panel_sweep, f_labelSize(panel_sweep_size))

                if (curModel.sweptHigh or curModel.sweptLow) and show_sweep_lines
                    color lineColor = curModel.dpurge ? dpurge_col : (curModel.sweptHigh ? sweep_bear_col : sweep_bull_col)
                    
                    float sweepLvl = curModel.sweptHigh ? prevModel.h : prevModel.l
                    int sweepStartBar = curModel.sweptHigh ? prevModel.h_bar : prevModel.l_bar
                    curModel.sweepLine := line.new(x1 = sweepStartBar, y1 = sweepLvl, x2 = curModel.endBar, y2 = sweepLvl, color = lineColor, width = 1, style = line.style_solid)
                    
                    string sweepTxt = (curModel.dpurge and show_dpurge) ? "D-Purge" : (curModel.sweptHigh ? "CRT H" : "CRT L")
                    curModel.sweepLabel := label.new(x = curModel.endBar, y = sweepLvl, text = sweepTxt, style = curModel.sweptHigh ? label.style_label_down : label.style_label_up, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))
                    
                    float targetLvl = curModel.sweptHigh ? prevModel.l : prevModel.h
                    int targetStartBar = curModel.sweptHigh ? prevModel.l_bar : prevModel.h_bar
                    curModel.targetLine := line.new(x1 = targetStartBar, y1 = targetLvl, x2 = curModel.endBar, y2 = targetLvl, color = lineColor, width = 1, style = line.style_solid)
                    string targetTxt = curModel.sweptHigh ? "CRT L" : "CRT H"
                    curModel.targetLabel := label.new(x = curModel.endBar, y = targetLvl, text = targetTxt, style = curModel.sweptHigh ? label.style_label_up : label.style_label_down, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))

                    alertSweepEvent := true
                    if curModel.dpurge
                        alertDpurgeEvent := true

                if show_model_labels and modelMode == "Model 2"
                    if curModel.sweptHigh or curModel.sweptLow
                        int   c1x = curModel.sweptHigh ? curModel.h_bar : curModel.l_bar
                        float c1y = curModel.sweptHigh ? curModel.h : curModel.l
                        curModel.c1Label := label.new(x = c1x, y = c1y, text = "C1", style = curModel.sweptHigh ? label.style_label_down : label.style_label_up, color = color.new(color.white, 100), textcolor = curModel.sweptHigh ? sweep_bear_col : sweep_bull_col, size = size.small)

                if modelMode == "Model 1"
                    if curModel.sweptHigh and showBearOB
                        int h_offset = bar_index - curModel.h_bar
                        float bearObLevel = h_offset >= 0 ? low[h_offset] : low
                        [lRef, lblRef] = f_drawRealtimeOB(false, bearObLevel, curModel.h_bar, bearOBColor, obLineWidth, obBarsToExtend, obTextSize, bearOBColor)
                        
                        array.push(obLinesArray, lRef)
                        array.push(obLabelsArray, lblRef)
                        if array.size(obLinesArray) > maxObCount
                            line.delete(array.shift(obLinesArray))
                        if array.size(obLabelsArray) > maxObCount
                            label.delete(array.shift(obLabelsArray))
                            
                    if curModel.sweptLow and showBullOB
                        int l_offset = bar_index - curModel.l_bar
                        float bullObLevel = l_offset >= 0 ? high[l_offset] : high
                        [lRef, lblRef] = f_drawRealtimeOB(true, bullObLevel, curModel.l_bar, bullOBColor, obLineWidth, obBarsToExtend, obTextSize, bullOBColor)
                        
                        array.push(obLinesArray, lRef)
                        array.push(obLabelsArray, lblRef)
                        if array.size(obLinesArray) > maxObCount
                            line.delete(array.shift(obLinesArray))
                        if array.size(obLabelsArray) > maxObCount
                            label.delete(array.shift(obLabelsArray))

                alertFormed := true

            if hide_overlap and not na(prevModel.rangeBox) and curModel.h <= prevModel.h and curModel.l >= prevModel.l
                box.delete(curModel.rangeBox)
                line.delete(curModel.separatorLine)

            modelHistory.push(curModel)
            prevModel := curModel

            while modelHistory.size() > history_lookback
                CRTModel oldModel = modelHistory.shift()
                f_deleteModel(oldModel)

        curModel := f_newModel(open, high, low, close, bar_index, neutral_box_col, crt_border_col, show_crt_box, sep_col, sep_width)

    else
        if high > curModel.h
            curModel.h          := high
            curModel.h_bar      := bar_index
            curModel.h_bar_open := open
        if low < curModel.l
            curModel.l          := low
            curModel.l_bar      := bar_index
            curModel.l_bar_open := open
        curModel.c := close
        if not na(curModel.rangeBox)
            box.set_top(curModel.rangeBox, curModel.h)
            box.set_bottom(curModel.rangeBox, curModel.l)
            box.set_right(curModel.rangeBox, bar_index)

        if modelMode == "Model 1" and not na(prevModel.h) and not na(prevModel.l)

            bool isBearishActive = rtSweptHigh or curModel.sweptHigh or prevModel.sweptHigh
            float bearInvalidLevel = not na(rtSweepHighLevel) ? rtSweepHighLevel : prevModel.h
            if isBearishActive and not na(bearInvalidLevel) and high >= bearInvalidLevel
                line.delete(curModel.sweepLine)
                line.delete(curModel.targetLine)
                label.delete(curModel.sweepLabel)
                label.delete(curModel.targetLabel)
                line.delete(prevModel.sweepLine)
                line.delete(prevModel.targetLine)
                label.delete(prevModel.sweepLabel)
                label.delete(prevModel.targetLabel)
                line.delete(rtObLine)
                label.delete(rtObLabel)
                curModel.sweepLine   := na
                curModel.targetLine  := na
                curModel.sweepLabel  := na
                curModel.targetLabel := na
                prevModel.sweepLine  := na
                prevModel.targetLine := na
                prevModel.sweepLabel := na
                prevModel.targetLabel:= na
                rtObLine             := na
                rtObLabel            := na
                rtSweptHigh          := false
                rtHighBreached       := false
                curModel.sweptHigh   := false
                prevModel.sweptHigh  := false
                rtSweepHighLevel     := na

                if array.size(obLinesArray) > 0
                    for i = array.size(obLinesArray) - 1 to 0
                        line.delete(array.get(obLinesArray, i))
                    array.clear(obLinesArray)
                if array.size(obLabelsArray) > 0
                    for i = array.size(obLabelsArray) - 1 to 0
                        label.delete(array.get(obLabelsArray, i))
                    array.clear(obLabelsArray)

            bool isBullishActive = rtSweptLow or curModel.sweptLow or prevModel.sweptLow
            float bullInvalidLevel = not na(rtSweepLowLevel) ? rtSweepLowLevel : prevModel.l
            if isBullishActive and not na(bullInvalidLevel) and low <= bullInvalidLevel
                line.delete(curModel.sweepLine)
                line.delete(curModel.targetLine)
                label.delete(curModel.sweepLabel)
                label.delete(curModel.targetLabel)
                line.delete(prevModel.sweepLine)
                line.delete(prevModel.targetLine)
                label.delete(prevModel.sweepLabel)
                label.delete(prevModel.targetLabel)
                line.delete(rtObLine)
                label.delete(rtObLabel)
                curModel.sweepLine   := na
                curModel.targetLine  := na
                curModel.sweepLabel  := na
                curModel.targetLabel := na
                prevModel.sweepLine  := na
                prevModel.targetLine := na
                prevModel.sweepLabel := na
                prevModel.targetLabel:= na
                rtObLine             := na
                rtObLabel            := na
                rtSweptLow           := false
                rtLowBreached        := false
                curModel.sweptLow    := false
                prevModel.sweptLow   := false
                rtSweepLowLevel      := na

                if array.size(obLinesArray) > 0
                    for i = array.size(obLinesArray) - 1 to 0
                        line.delete(array.get(obLinesArray, i))
                    array.clear(obLinesArray)
                if array.size(obLabelsArray) > 0
                    for i = array.size(obLabelsArray) - 1 to 0
                        label.delete(array.get(obLabelsArray, i))
                    array.clear(obLabelsArray)

            if high > prevModel.h and (biasFilter == "Neutral" or biasFilter == "Bearish")
                rtHighBreached := true
            if low < prevModel.l and (biasFilter == "Neutral" or biasFilter == "Bullish")
                rtLowBreached := true
                
            if rtHighBreached and not rtSweptHigh and close < prevModel.h
                rtSweptHigh := true
                rtSweepHighLevel := curModel.h
                if show_sweep_lines
                    float sweepLvl = prevModel.h
                    float targetLvl = prevModel.l
                    curModel.sweepLine := line.new(x1 = prevModel.h_bar, y1 = sweepLvl, x2 = bar_index, y2 = sweepLvl, color = sweep_bear_col, width = 1, style = line.style_solid)
                    curModel.sweepLabel := label.new(x = bar_index, y = sweepLvl, text = "CRT H", style = label.style_label_down, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))
                    
                    curModel.targetLine := line.new(x1 = prevModel.l_bar, y1 = targetLvl, x2 = bar_index, y2 = targetLvl, color = sweep_bear_col, width = 1, style = line.style_solid)
                    curModel.targetLabel := label.new(x = bar_index, y = targetLvl, text = "CRT L", style = label.style_label_up, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))
                
                if showBearOB
                    int h_offset = bar_index - curModel.h_bar
                    float bearObLevel = h_offset >= 0 ? low[h_offset] : low
                    
                    if not na(rtObLine)
                        line.delete(rtObLine)
                    if not na(rtObLabel)
                        label.delete(rtObLabel)
                        
                    [lRef, lblRef] = f_drawRealtimeOB(false, bearObLevel, curModel.h_bar, bearOBColor, obLineWidth, obBarsToExtend, obTextSize, bearOBColor)
                    rtObLine := lRef
                    rtObLabel := lblRef
                
                alertSweepEvent := true

            if rtLowBreached and not rtSweptLow and close > prevModel.l
                rtSweptLow := true
                rtSweepLowLevel := curModel.l
                if show_sweep_lines
                    float sweepLvl = prevModel.l
                    float targetLvl = prevModel.h
                    curModel.sweepLine := line.new(x1 = prevModel.l_bar, y1 = sweepLvl, x2 = bar_index, y2 = sweepLvl, color = sweep_bull_col, width = 1, style = line.style_solid)
                    curModel.sweepLabel := label.new(x = bar_index, y = sweepLvl, text = "CRT L", style = label.style_label_up, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))
                    
                    curModel.targetLine := line.new(x1 = prevModel.h_bar, y1 = targetLvl, x2 = bar_index, y2 = targetLvl, color = sweep_bull_col, width = 1, style = line.style_solid)
                    curModel.targetLabel := label.new(x = bar_index, y = targetLvl, text = "CRT H", style = label.style_label_down, color = color.new(color.white, 100), textcolor = label_text_col, size = f_labelSize(label_size_inp))
                
                if showBullOB
                    int l_offset = bar_index - curModel.l_bar
                    float bullObLevel = l_offset >= 0 ? high[l_offset] : high
                    
                    if not na(rtObLine)
                        line.delete(rtObLine)
                    if not na(rtObLabel)
                        label.delete(rtObLabel)
                        
                    [lRef, lblRef] = f_drawRealtimeOB(true, bullObLevel, curModel.l_bar, bullOBColor, obLineWidth, obBarsToExtend, obTextSize, bullOBColor)
                    rtObLine := lRef
                    rtObLabel := lblRef
                
                alertSweepEvent := true

        if modelMode == "Model 2" and not na(prevModel.startBar) and not curModel.cisdDone and show_cisd and (prevModel.sweptHigh or prevModel.sweptLow) and not na(prevModel.manipOpen)
            if prevModel.sweptHigh and close < prevModel.manipOpen
                curModel.cisdDone  := true
                curModel.cisdBar   := bar_index
                curModel.cisdLevel := prevModel.manipOpen
                curModel.cisdLine  := line.new(x1 = prevModel.manipBar, y1 = prevModel.manipOpen, x2 = bar_index, y2 = prevModel.manipOpen, color = cisd_bear_col, width = 1)
                curModel.cisdLabel := label.new(x = bar_index, y = prevModel.manipOpen, text = "CISD 🡇", style = label.style_label_left, color = color.new(color.white, 100), textcolor = cisd_bear_col, size = size.small)
                if show_model_labels
                    curModel.c3Label := label.new(x = bar_index, y = close, text = "C3", style = label.style_label_down, color = color.new(color.white, 100), textcolor = sweep_bear_col, size = size.small)
            else if prevModel.sweptLow and close > prevModel.manipOpen
                curModel.cisdDone  := true
                curModel.cisdBar   := bar_index
                curModel.cisdLevel := prevModel.manipOpen
                curModel.cisdLine  := line.new(x1 = prevModel.manipBar, y1 = prevModel.manipOpen, x2 = bar_index, y2 = prevModel.manipOpen, color = cisd_bull_col, width = 1)
                curModel.cisdLabel := label.new(x = bar_index, y = prevModel.manipOpen, text = "CISD 🡅", style = label.style_label_left, color = color.new(color.white, 100), textcolor = cisd_bull_col, size = size.small)
                if show_model_labels
                    curModel.c3Label := label.new(x = bar_index, y = close, text = "C3", style = label.style_label_up, color = color.new(color.white, 100), textcolor = sweep_bull_col, size = size.small)

        if not na(prevModel.rangeBox) and prevModel.status != "Success" and prevModel.status != "Invalidated"
            color newStatusCol = na
            if prevModel.bias == "Bearish" and high > prevModel.h
                prevModel.status := "Invalidated"
                newStatusCol := red_col
                alertInvalidated := true
            else if prevModel.bias == "Bullish" and low < prevModel.l
                prevModel.status := "Invalidated"
                newStatusCol := red_col
                alertInvalidated := true
            else if prevModel.bias == "Bearish" and low <= prevModel.l
                prevModel.status := "Success"
                newStatusCol := green_col
                alertSuccess := true
            else if prevModel.bias == "Bullish" and high >= prevModel.h
                prevModel.status := "Success"
                newStatusCol := green_col
                alertSuccess := true

            if not na(newStatusCol)
                box.set_border_color(prevModel.rangeBox, show_crt_box ? newStatusCol : color.new(color.white, 100))
                if sync_right_panel
                    f_syncCandlePanel(htf1, prevModel.startBar, newStatusCol, false, false, color.gray, false, size.tiny)

    newPeriod_smt = ta.change(time(htf_str)) != 0
    if enableSMT and bar_index > last_bar_index - 4500 and htf_str != ""
        sPair = smtAutoMode ? getSMTPair() : str.tostring(smtManualPair)
        if sPair != ""
            [sH_htf, sL_htf, sH1_htf, sL1_htf] = request.security(sPair, htf_str, [high, low, high[1], low[1]], lookahead = barmerge.lookahead_off)
            [sH_piv, sL_piv] = request.security(sPair, timeframe.period, [high[3], low[3]], lookahead = barmerge.lookahead_off)
            
            chartPivDir = not na(ta.pivothigh(3, 3)) ? 1 : not na(ta.pivotlow(3, 3)) ? -1 : 0
            
            [htfClose, htfOpen, htfHigh, htfLow, htfHigh1, htfLow1] = request.security(syminfo.tickerid, htf_str, [close, open, high, low, high[1], low[1]], lookahead = barmerge.lookahead_off)
            
            bearD_htf = (not na(htfClose) and htfClose < htfOpen) and htfHigh < htfHigh1 and sH_htf > sH1_htf
            bullD_htf = (not na(htfClose) and htfClose > htfOpen) and htfLow > htfLow1 and sL_htf < sL1_htf
            
            if high >= currCH_smt or na(currCH_smt)
                currCH_smt := high, currCHT_smt := time
            if low <= currCL_smt or na(currCL_smt)
                currCL_smt := low, currCLT_smt := time
                
            if newPeriod_smt
                if bearD_htf[1]
                    smtArray.push(SMT.new("BEARISH", sPair, prevCH_smt, currCH_smt, prevCHT_smt, currCHT_smt, na, na, na, na, true, bar_index, na, na))
                if bullD_htf[1]
                    smtArray.push(SMT.new("BULLISH", sPair, na, na, na, na, prevCL_smt, currCL_smt, prevCLT_smt, currCLT_smt, true, bar_index, na, na))
                prevCH_smt := currCH_smt, prevCHT_smt := currCHT_smt
                prevCL_smt := currCL_smt, prevCLT_smt := currCLT_smt
                currCH_smt := high, currCHT_smt := time
                currCL_smt := low, currCLT_smt := time
                line.delete(liveBearSmtL), liveBearSmtL := na
                line.delete(liveBullSmtL), liveBullSmtL := na

            if chartPivDir == 1 
                if not na(chartPivotHigh)
                    prevChartPivotHigh := chartPivotHigh
                    prevChartPivotHighTime := chartPivotHighTime
                chartPivotHigh := high[3], chartPivotHighTime := time[3]
                
                if not na(smtPivotHigh)
                    prevSmtPivotHigh := smtPivotHigh
                smtPivotHigh := sH_piv
                
                if not na(prevChartPivotHigh) and chartPivotHigh > prevChartPivotHigh and smtPivotHigh < prevSmtPivotHigh
                    smtArray.push(SMT.new("BEARISH", sPair, prevChartPivotHigh, chartPivotHigh, prevChartPivotHighTime, chartPivotHighTime, na, na, na, na, true, bar_index, na, na))

            if chartPivDir == -1 
                if not na(chartPivotLow)
                    prevChartPivotLow := chartPivotLow
                    prevChartPivotLowTime := chartPivotLowTime
                chartPivotLow := low[3], chartPivotLowTime := time[3]
                
                if not na(smtPivotLow)
                    prevSmtPivotLow := smtPivotLow
                smtPivotLow := sL_piv
                
                if not na(prevChartPivotLow) and chartPivotLow < prevChartPivotLow and smtPivotLow > prevSmtPivotLow
                    smtArray.push(SMT.new("BULLISH", sPair, na, na, na, na, prevChartPivotLow, chartPivotLow, prevChartPivotLowTime, chartPivotLowTime, true, bar_index, na, na))

    if htf1.settings.show
        htf1.Monitor().Update(settings.offset)

    if barstate.islast
        if enableSMT and bar_index > last_bar_index - 4500
            if smtKeepOnlyLatest and array.size(smtArray) > 0
                int latestBearIdx = -1
                int latestBullIdx = -1
                int maxBearBar = -1
                int maxBullBar = -1
                
                for i = 0 to array.size(smtArray) - 1
                    s_check = array.get(smtArray, i)
                    if s_check.smtType == "BEARISH" and s_check.creationBar > maxBearBar
                        maxBearBar := s_check.creationBar
                        latestBearIdx := i
                    if s_check.smtType == "BULLISH" and s_check.creationBar > maxBullBar
                        maxBullBar := s_check.creationBar
                        latestBullIdx := i
                
                for i = array.size(smtArray) - 1 to 0
                    if (array.get(smtArray, i).smtType == "BEARISH" and i != latestBearIdx) or (array.get(smtArray, i).smtType == "BULLISH" and i != latestBullIdx)
                        s_del = array.remove(smtArray, i)
                        line.delete(s_del.smtLine)

            int szL = array.size(smtLabels)
            if szL > 0
                for i = 0 to szL - 1
                    label.delete(array.get(smtLabels, i))
                array.clear(smtLabels)

            int szA = array.size(smtArray)
            if szA > 0
                for i = szA - 1 to 0
                    s = array.get(smtArray, i)
                    if bar_index - s.creationBar > 2500
                        line.delete(s.smtLine)
                        array.remove(smtArray, i)
                        continue
                    
                    isB = s.smtType == "BEARISH"
                    c = color.rgb(204, 162, 9) 
                    
                    p1T = isB ? s.prevChartHighTime : s.prevChartLowTime
                    p1P = isB ? s.prevChartHigh : s.prevChartLow
                    p2T = isB ? s.currChartHighTime : s.currChartLowTime
                    p2P = isB ? s.currChartHigh : s.currChartLow
                    
                    if not na(p1T) and not na(p2T)
                        if na(s.smtLine)
                            s.smtLine := line.new(p1T, p1P, p2T, p2P, xloc=xloc.bar_time, color=c, width=smtLineWidth)
                        else
                            line.set_xy1(s.smtLine, p1T, p1P)
                            line.set_xy2(s.smtLine, p2T, p2P)
                            line.set_color(s.smtLine, c)

                        if show_crt_labels
                            string cleanName = s.smtPair
                            if str.contains(cleanName, ":")
                                cleanName := array.get(str.split(cleanName, ":"), array.size(str.split(cleanName, ":")) - 1)
                            cleanName := str.replace_all(cleanName, ".P", ""), cleanName := str.replace_all(cleanName, "1!", "")
                            
                            labelStyle = isB ? label.style_label_down : label.style_label_up
                            
                            newL = label.new((p1T+p2T)/2, (p1P+p2P)/2, cleanName, 
                                             color=color.new(color.white, 100), 
                                             textcolor=c, 
                                             style=labelStyle, 
                                             size=size.small, 
                                             xloc=xloc.bar_time)
                            array.push(smtLabels, newL)
                            
        int panelCount   = htf1.candles.size()
        int sizeC        = panelCount
        int centerX      = bar_index + settings.offset + int(((settings.width + settings.buffer) * math.max(sizeC - 1, 0)) / 2) + int(settings.width / 2)
        
        float highestH   = f_candlesHigh(htf1.candles)
        float lowestL    = na
        if sizeC > 0
            for i = 0 to sizeC - 1
                Candle cnd = htf1.candles.get(i)
                lowestL := na(lowestL) ? cnd.l : math.min(lowestL, cnd.l)

        float baseHigh   = na(highestH) ? high : highestH
        float baseLow    = na(lowestL) ? low : lowestL

        string cd = f_countdownClean(htf_str)
        string tfFriendly = f_friendlyTF(htf_str)
        string topLabelTxt = tfFriendly + "\n(" + cd + ")"

        if show_big_tf_label
            if na(htf1.tfLabel)
                htf1.tfLabel := label.new(x = centerX, y = baseHigh, text = topLabelTxt, style = label.style_label_down, color = color.new(color.white, 100), textcolor = big_tf_color, size = f_labelSize(big_tf_size_i), text_font_family = font.family_monospace)
            else
                label.set_xy(htf1.tfLabel, centerX, baseHigh)
                label.set_text(htf1.tfLabel, topLabelTxt)

        if show_info_text
            color biasCol2 = globalBias == "Bullish" ? green_col : globalBias == "Bearish" ? red_col : grey_col
            string biasTxt = "Bias: " + globalBias

            if na(biasTextLabel)
                biasTextLabel := label.new(x = centerX, y = baseLow, text = biasTxt, style = label.style_label_up, color = color.new(color.white, 100), textcolor = biasCol2, size = f_labelSize(info_text_size_i), text_font_family = font.family_monospace)
            else
                label.set_xy(biasTextLabel, centerX, baseLow)
                label.set_text(biasTextLabel, biasTxt)
                label.set_textcolor(biasTextLabel, biasCol2)

var table dash = table.new(position = f_dashPos(dash_position_i), columns = 1, rows = 5, bgcolor = color.white, border_width = 1, border_color = color.black, frame_width = 1, frame_color = color.black)

if show_dashboard and barstate.islast
    string txtSize = f_dashSize(dash_size_i)
    
    string chartTFFriendly = f_friendlyTF(timeframe.period)
    string htfTFFriendly   = f_friendlyTF(htf_str)
    
    string titleText = "CRT Model (" + chartTFFriendly + "-" + htfTFFriendly + ")"
    color headerBg = color.new(#131722, 0)
    table.cell(dash, 0, 0, titleText, text_size = txtSize, text_color = color.white, bgcolor = headerBg, text_font_family = font.family_monospace)
    
    string modeText = modelMode
    table.cell(dash, 0, 1, modeText, text_size = txtSize, text_color = color.black, bgcolor = color.white, text_font_family = font.family_monospace)
    
    string biasText = "Bias: " + globalBias
    color biasCol = globalBias == "Bullish" ? green_col : globalBias == "Bearish" ? red_col : grey_col
    table.cell(dash, 0, 2, biasText, text_size = txtSize, text_color = biasCol, bgcolor = color.white, text_font_family = font.family_monospace)
    
    string smtPairText = syminfo.ticker
    if enableSMT
        string sPair = smtAutoMode ? getSMTPair() : str.tostring(smtManualPair)
        if sPair != ""
            if str.contains(sPair, ":")
                sPair := array.get(str.split(sPair, ":"), array.size(str.split(sPair, ":")) - 1)
            sPair := str.replace_all(sPair, ".P", ""), sPair := str.replace_all(sPair, "1!", "")
            smtPairText := syminfo.ticker + " / " + sPair + (smtAutoMode ? " (Auto)" : " (Manual)")
    table.cell(dash, 0, 3, smtPairText, text_size = txtSize, text_color = color.black, bgcolor = color.white, text_font_family = font.family_monospace)
    
    string dateText = str.format("{0,date,d MMM yyyy}", timenow)
    table.cell(dash, 0, 4, dateText, text_size = txtSize, text_color = color.black, bgcolor = color.white, text_font_family = font.family_monospace)

alertcondition(alertFormed, title = "Model Formed", message = "CRT Model Formed - {{ticker}} - {{interval}}")
alertcondition(alertSuccess, title = "Model Successful", message = "CRT Model Successful - {{ticker}} - {{interval}}")
alertcondition(alertInvalidated, title = "Model Invalidated", message = "CRT Model Invalidated - {{ticker}} - {{interval}}")
alertcondition(alertSweepEvent, title = "Model Sweep", message = "CRT Sweep Detected - {{ticker}} - {{interval}}")
alertcondition(alertDpurgeEvent, title = "Model D-Purge Sweep", message = "CRT Double Purge Detected - {{ticker}} - {{interval}}")