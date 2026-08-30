//@version=6
indicator("CISD PRO+", shorttitle = "CISD PRO+", overlay = true, max_lines_count = 500, max_labels_count = 500, max_boxes_count = 500, max_bars_back = 500, max_polylines_count = 100)

colors_transparent = color.new(color.white,100)
colors_black_default = color.black

gp_cisd = "CISD Integration Settings"
show_cisd = input.bool(true, "CISD Lines         ", inline = "cisd_1", group = gp_cisd)
cisd_maxLines = input.int(5, "Max", minval = 1, maxval = 10, inline = "cisd_1", group = gp_cisd)
cisd_bull_color = input.color(color.new(#5b9cf6, 0), "Bull", inline = "cisd_1", group = gp_cisd)
cisd_bear_color = input.color(color.new(#f77c80, 0), "Bear", inline = "cisd_1", group = gp_cisd)
cisd_line_width = input.int(2, "Width", minval = 1, maxval = 5, inline = "cisd_2", group = gp_cisd)
cisd_line_style = input.string("Solid", "Style", options = ["Solid", "Dashed", "Dotted"], inline = "cisd_2", group = gp_cisd)

show_projections = input.bool(true, "Projections  ", inline = "cisd_3", group = gp_cisd)
projection_levels = input.string("1,2,2.5", "Levels", inline = "cisd_3", group = gp_cisd)
proj_size = input.string("Small", "Size", options = ["Tiny", "Small", "Normal", "Large"], inline = "cisd_3", group = gp_cisd)
proj_color = input.color(color.new(#787B86, 0), "Color", inline = "cisd_3", group = gp_cisd)

dot_size = input.string("Small", "Dot Size", options = ["Tiny", "Small", "Normal", "Large"], inline = "cisd_4", group = gp_cisd)

show_cisd_level = input.bool(true, "Level Label  ", inline = "cisd_5", group = gp_cisd)
cisd_level_size = input.string("Small", "Size", options = ["Tiny", "Small", "Normal", "Large"], inline = "cisd_5", group = gp_cisd)
cisd_line_extend = input.int(50, "Line Length (bars right)", minval = 0, maxval = 500, tooltip = "How far past the CISD confirmation bar the line (and its price label) is drawn. Raise for a longer line, lower for a shorter one.", group = gp_cisd)

gp_conf = "CISD Confluence Filters"
filter_session = input.bool(false, "Filter by Session High/Low", group = gp_conf)
filter_fvg = input.bool(true, "Filter by HTF FVG", group = gp_conf)

gp_sessions = "Session Levels"
showSessions = input.bool(false, "Enable Session Levels", group = gp_sessions)
session_tz = input.string('America/New_York', "Timezone", group = gp_sessions)
max_sessions = input.int(1, "Max Past Sessions to Show", minval = 1, maxval = 20, group = gp_sessions)
session_style = input.string('⎯⎯⎯', 'Line Style', inline = 'ses_st', group = gp_sessions, options = ['⎯⎯⎯', '----', '····'])
session_width = input.int(1, 'Line Width', inline = 'ses_st', group = gp_sessions, minval = 1, maxval = 5)
session_lbl = input.string("Small", "Label Size", options = ["Hide", "Tiny", "Small", "Normal", "Large", "Huge", "Auto"], group = gp_sessions)

use_asia = input.bool(false, "Asia          ", inline = "ses_as", group = gp_sessions)
asia_time = input.session("1900-2200", "", inline = "ses_as", group = gp_sessions)
asia_color = input.color(color.new(#5b9cf6, 20), "", inline = "ses_as", group = gp_sessions)

use_london = input.bool(false, "London      ", inline = "ses_lo", group = gp_sessions)
london_time = input.session("0200-0500", "", inline = "ses_lo", group = gp_sessions)
london_color = input.color(color.new(#f77c80, 20), "", inline = "ses_lo", group = gp_sessions)

use_ny = input.bool(false, "New York   ", inline = "ses_ny", group = gp_sessions)
ny_time = input.session("0700-1000", "", inline = "ses_ny", group = gp_sessions)
ny_color = input.color(color.new(#2dd06e, 20), "", inline = "ses_ny", group = gp_sessions)

use_lc = input.bool(false, "London Close", inline = "ses_lc", group = gp_sessions)
lc_time = input.session("1000-1200", "", inline = "ses_lc", group = gp_sessions)
lc_color = input.color(color.new(#c77dff, 20), "", inline = "ses_lc", group = gp_sessions)

gp_alerts = "Alert Settings"
alert_sessions = input.bool(false, "Alert on Session High/Low Break", group = gp_alerts)

gp_tf = "HTF FVG Settings"
autoMode = input.bool(true, 'Auto Timeframe Mode', tooltip = 'Automatically select HTF based on current chart timeframe.', group = gp_tf)
manualHTF = input.timeframe('D', 'Manual HTF', tooltip = 'Manual HTF selection (only used when Auto Mode is disabled)', group = gp_tf)

gp_style = "FVG Visual Settings"
showBullishFVG = input.bool(true, "Bullish FVG   ", inline = "fvg_1", group = gp_style)
bullishFVGColor = input.color(color.new(#808080, 90), "Color", inline = "fvg_1", group = gp_style)

showBearishFVG = input.bool(true, "Bearish FVG  ", inline = "fvg_2", group = gp_style)
bearishFVGColor = input.color(color.new(#808080, 90), "Color", inline = "fvg_2", group = gp_style)

showMidlines = input.bool(true, "Show Midline ", inline = "fvg_3", group = gp_style)
midlineColor = input.color(color.new(color.gray, 75), "Color", inline = "fvg_3", group = gp_style)
midlineStyle = input.string('····', 'Style', inline = 'fvg_3', group = gp_style, options = ['⎯⎯⎯', '----', '····'])
midlineWidth = input.int(1, 'Width', inline = 'fvg_3', group = gp_style, minval = 1, maxval = 5)

showLabels = input.bool(true, "Show Labels   ", inline = "fvg_4", group = gp_style)
labelSize = input.string("Small", "Size", options = ["Tiny", "Small", "Normal", "Large", "Huge", "Auto"], inline = "fvg_4", group = gp_style)
labelTextColor = input.color(color.gray, "Color", inline = "fvg_4", group = gp_style)

showFVGBoxes = input.bool(true, 'Show FVG Boxes', group = gp_style)
maxFVGs = input.int(5, 'Max Mitigated FVGs to Keep', minval = 1, maxval = 100, group = gp_style)

gp_tbl = "Info Table Settings"
show_table = input.bool(true, "Show Info Table", group = gp_tbl)
table_pos_str = input.string("Top Right", "Table Position", options=["Top Right", "Top Left", "Top Center", "Bottom Right", "Bottom Left", "Bottom Center", "Middle Right", "Middle Left", "Center"], group=gp_tbl)
bg_col = input.color(color.new(#e2e2e2, 10), "Background Color", group=gp_tbl)
border_col = input.color(color.black, "Border Color", group=gp_tbl)
text_col = input.color(color.black, "Text Color", group=gp_tbl)
frame_w = input.int(2, "Outer Frame Width", minval=0, maxval=10, group=gp_tbl)
border_w = input.int(1, "Inner Border Width", minval=0, maxval=10, group=gp_tbl)
ticker_size_str = input.string("Large", "Ticker Text Size", options=["Tiny", "Small", "Normal", "Large", "Huge"], group=gp_tbl)
label_size_str = input.string("Small", "Label Text Size", options=["Tiny", "Small", "Normal", "Large", "Huge"], group=gp_tbl)

type FVG
    box fvgBox
    line midline
    label fvgLabel
    float top
    float bottom
    bool isBullish
    int startTime
    bool mitigated
    int mitigationTime

var FVG[] activeFVGs = array.new<FVG>()

type cisdSetup
    line cisdLine
    label baseDot
    label levelLabel
    array<label> projection_labels
    float basePrice
    bool bullish

var array<cisdSetup> cisd_setups = array.new<cisdSetup>()

var int market_bias = 0
var float cisd_level = na
var int cisd_start_bar = na
var float swing_pivot = na
var int swing_pivot_bar = na

var bool support_touch = false
var bool resistance_touch = false

method text_size_cisd(string size) =>
    out = switch size
        "Tiny" => size.tiny
        "Small" => size.small
        "Normal" => size.normal
        "Large" => size.large
        => size.auto
    out

getCurrentLegExtreme(int bias) =>
    int temp_shift = 0
    float extreme = open[0]
    int candle_type = close[0] > open[0] ? 1 : close[0] < open[0] ? -1 : 0
    float out_extreme = extreme
    int out_shift = bar_index
    bool early_exit = candle_type == 0 or candle_type != bias
    if not early_exit
        for i = 1 to 500
            candle_type := close[i] > open[i] ? 1 : close[i] < open[i] ? -1 : 0
            if candle_type == 0
                continue
            bool mood_match = candle_type == bias
            if not mood_match
                break
            temp_shift := i
            if bias == 1
                if open[i] < extreme
                    extreme := open[i]
            else
                if open[i] > extreme
                    extreme := open[i]
        int extreme_shift = na
        for k = 0 to temp_shift
            if open[k] == extreme
                extreme_shift := k
                break
        out_extreme := extreme
        out_shift := na(extreme_shift) ? bar_index : bar_index - extreme_shift
    [out_extreme, out_shift]

getPreviousLegExtreme(int bias) =>
    int i = 1
    bool artifact_found = false
    int max_shift = na
    float extreme = na
    for j = 1 to 500
        i := j
        int candle_type = close[i] > open[i] ? 1 : close[i] < open[i] ? -1 : 0
        if candle_type == 0
            continue
        bool is_correct_era = candle_type == bias
        if not artifact_found
            if is_correct_era
                artifact_found := true
                max_shift := i
                extreme := open[i]
        else
            if not is_correct_era
                break
            max_shift := i
            if bias == 1
                if open[i] < extreme
                    extreme := open[i]
            else
                if open[i] > extreme
                    extreme := open[i]
    int extreme_shift = na
    if not na(max_shift)
        for k = 1 to max_shift
            if open[k] == extreme
                extreme_shift := k
                break
    [extreme, na(extreme_shift) ? na : bar_index - extreme_shift]

type kz
    string _title
    array<box> _box
    array<line> _hi_line
    array<line> _lo_line
    array<label> _hi_label
    array<label> _lo_label
    array<bool> _hi_valid
    array<bool> _lo_valid
    float prevHigh = na
    float prevLow = na
    bool prevHighActive = false
    bool prevLowActive = false

type kz_helper
    kz _kz
    string session
    string tz
    color c
    string hi_txt
    string lo_txt

new_kz_state(string title) =>
    kz.new(
        _title = title,
        _box = array.new_box(),
        _hi_line = array.new_line(),
        _lo_line = array.new_line(),
        _hi_label = array.new_label(),
        _lo_label = array.new_label(),
        _hi_valid = array.new_bool(),
        _lo_valid = array.new_bool())

initKZ() =>
    array<kz_helper> res = array.new<kz_helper>()
    if use_asia
        res.push(kz_helper.new(_kz = new_kz_state('Asia'), session = asia_time, tz = session_tz, c = asia_color, hi_txt = 'AS.H', lo_txt = 'AS.L'))
    if use_london
        res.push(kz_helper.new(_kz = new_kz_state('London'), session = london_time, tz = session_tz, c = london_color, hi_txt = 'LO.H', lo_txt = 'LO.L'))
    if use_ny
        res.push(kz_helper.new(_kz = new_kz_state('New York'), session = ny_time, tz = session_tz, c = ny_color, hi_txt = 'NY.H', lo_txt = 'NY.L'))
    if use_lc
        res.push(kz_helper.new(_kz = new_kz_state('London Close'), session = lc_time, tz = session_tz, c = lc_color, hi_txt = 'LC.H', lo_txt = 'LC.L'))
    res

var array<kz_helper> key_zones = initKZ()

// Chart TF -> HTF map, kept strictly increasing so a bigger chart TF never
// resolves to a smaller HTF than a smaller chart TF (previous version could
// jump backward, e.g. a 6H chart mapped to "4D" while a 2H chart mapped to "1W").
getHTFPeriod() =>
    int tfSeconds = timeframe.in_seconds()
    string result = "M"
    if tfSeconds <= 60          // <= 1min
        result := "15"
    else if tfSeconds <= 300    // <= 5min
        result := "60"
    else if tfSeconds <= 900    // <= 15min
        result := "240"
    else if tfSeconds <= 3600   // <= 1H  (30min-1H swing entries -> Daily HTF)
        result := "1D"
    else if tfSeconds <= 14400  // <= 4H  -> Weekly HTF
        result := "1W"
    else if tfSeconds <= 86400  // <= 1D  -> Monthly HTF
        result := "1M"
    else
        result := "3M"
    result

formatTimeframe(string tf) =>
    seconds = timeframe.in_seconds(tf)
    if seconds < 60
        str.tostring(seconds) + "S"
    else if (seconds / 60) < 60
        str.tostring(math.round(seconds/60)) + "M"
    else if (seconds/60/60) < 24
        str.tostring(math.round(seconds/60/60)) + "H"
    else if (seconds/60/60/24) < 7
        str.tostring(math.round(seconds/60/60/24)) + "D"
    else if (seconds/60/60/24/7) < 4
        str.tostring(math.round(seconds/60/60/24/7)) + "W"
    else if (seconds/60/60/24/30) < 12
        str.tostring(math.round(seconds/60/60/24/30)) + "M"
    else
        str.tostring(math.round(seconds/60/60/24/365)) + "Y"

detectFVG() =>
    bool hasBearishFVG = false
    bool hasBullishFVG = false
    float bearishBottom = na
    float bearishTop = na
    float bullishBottom = na
    float bullishTop = na
    int htfTimeVal = na
    int htfTime2Val = na

    if barstate.isconfirmed
        float htfHigh0 = high[0]
        float htfLow0 = low[0]
        float htfHigh2 = high[2]
        float htfLow2 = low[2]
        int htfTime0 = time[0]

        if not na(htfLow2) and htfHigh0 < htfLow2
            hasBearishFVG := true
            bearishBottom := htfHigh0
            bearishTop := htfLow2
            htfTimeVal := htfTime0
            htfTime2Val := time[2]
        else
            hasBearishFVG := false

        if not na(htfHigh2) and htfLow0 > htfHigh2
            hasBullishFVG := true
            bullishBottom := htfHigh2
            bullishTop := htfLow0
            htfTimeVal := htfTime0
            htfTime2Val := time[2]
        else
            hasBullishFVG := false

    [hasBearishFVG, hasBullishFVG, bearishBottom, bearishTop, bullishBottom, bullishTop, htfTimeVal, htfTime2Val]

method text_size(string size) =>
    out = switch size
        "Tiny" => size.tiny
        "Small" => size.small
        "Normal" => size.normal
        "Large" => size.large
        "Huge" => size.huge
        "Auto" => size.auto
    out

method line_style(string line) =>
    out = switch line
        '⎯⎯⎯' => line.style_solid
        '----' => line.style_dashed
        '····' => line.style_dotted

method font_style(string font) =>
    out = switch font
        'Default' => font.family_default
        'Monospace' => font.family_monospace

to_friendly_tf(tf_in) =>
    switch tf_in
        "1" => "1m"
        "5" => "5m"
        "15" => "15m"
        "30" => "30m"
        "60" => "1H"
        "120" => "2H"
        "180" => "3H"
        "240" => "4H"
        "1440" => "1D"
        "D" => "1D"
        "W" => "1W"
        "M" => "1M"
        => tf_in

get_position(pos_str) =>
    switch pos_str
        "Top Right" => position.top_right
        "Top Left" => position.top_left
        "Top Center" => position.top_center
        "Bottom Right" => position.bottom_right
        "Bottom Left" => position.bottom_left
        "Bottom Center" => position.bottom_center
        "Middle Right" => position.middle_right
        "Middle Left" => position.middle_left
        "Center" => position.middle_center
        => position.top_right

get_size(size_str) =>
    switch size_str
        "Tiny" => size.tiny
        "Small" => size.small
        "Normal" => size.normal
        "Large" => size.large
        "Huge" => size.huge
        => size.normal

method session_begins(string tf, string ses, string tz) =>
    ta.change(time(tf, ses, na(tz) ? "" : tz))!= 0

method in_session(string tf, string ses, string tz) =>
    t = time(tf, ses, na(tz) ? "" : tz)
    ct = time_close(tf, ses, na(tz) ? "" : tz)
    not na(t) and not na(ct)

is_bullish_candle(float c, float o, float h, float l) =>
    if c == o
        math.abs(o - h) < math.abs(o - l)
    else
        c > o

del_kz(kz k) =>
    if k._box.size() > max_sessions
        box.delete(k._box.pop())
    if k._hi_line.size() > max_sessions
        line.delete(k._hi_line.pop())
        line.delete(k._lo_line.pop())
        array.pop(k._hi_valid)
        array.pop(k._lo_valid)
    if k._hi_label.size() > max_sessions
        label.delete(k._hi_label.pop())
        label.delete(k._lo_label.pop())

adjust_in_kz(kz kz, bool t) =>
    if t
        kzBox0 = kz._box.get(0)
        kzBox0.set_right(time)
        newTop = math.max(kzBox0.get_top(), high)
        kzBox0.set_top(newTop)
        newBottom = math.min(kzBox0.get_bottom(), low)
        kzBox0.set_bottom(newBottom)

        if kz._hi_line.size() > 0
            kzHiLine0 = kz._hi_line.get(0)
            kzHiLine0.set_x2(bar_index)
            if high > kzHiLine0.get_y1()
                kzHiLine0.set_xy1(bar_index, high)
                kzHiLine0.set_xy2(bar_index, high)

            kzLoLine0 = kz._lo_line.get(0)
            kzLoLine0.set_x2(bar_index)
            if low < kzLoLine0.get_y1()
                kzLoLine0.set_xy1(bar_index, low)
                kzLoLine0.set_xy2(bar_index, low)

        if kz._hi_label.size() > 0
            if high > kz._hi_label.get(0).get_y()
                kz._hi_label.get(0).set_xy(time, high)
            if low < kz._lo_label.get(0).get_y()
                kz._lo_label.get(0).set_xy(time, low)

adjust_out_kz(kz kz, bool t, bool t_prev) =>
    boxCount = kz._box.size()
    if not t and boxCount > 0
        for i = 0 to boxCount - 1 by 1
            if i == 0
                if kz._hi_valid.get(i)
                    kz._hi_line.get(i).set_x2(bar_index)
                    kz._hi_label.get(i).set_x(time)

                    if high >= kz._hi_line.get(i).get_y1()
                        kz._hi_valid.set(i, false)
                        kz._hi_label.get(i).set_x(time[3])
                        if alert_sessions
                            alert(kz._title + ' High Touched', alert.freq_once_per_bar)

                if kz._lo_valid.get(i)
                    kz._lo_line.get(i).set_x2(bar_index)
                    kz._lo_label.get(i).set_x(time)

                    if low <= kz._lo_line.get(i).get_y1()
                        kz._lo_valid.set(i, false)
                        kz._lo_label.get(i).set_x(time[3])
                        if alert_sessions
                            alert(kz._title + " Low Touched", alert.freq_once_per_bar)
            else
                break

method manage_kz(kz_helper this) =>
    kz kz = this._kz
    c = this.c
    string hi_txt = this.hi_txt
    string lo_txt = this.lo_txt

    if timeframe.isintraday
        t = not na(time("", this.session, this.tz))
        t_prev = not na(time("", this.session, this.tz, bars_back = 1))

        if t_prev and not t and kz._hi_line.size() > 0
            kz.prevHigh := kz._hi_line.get(0).get_y1()
            kz.prevLow := kz._lo_line.get(0).get_y1()
            kz.prevHighActive := true
            kz.prevLowActive := true

        if t and not t_prev
            kz._box.unshift(box.new(time, high, time, low, xloc = xloc.bar_time, border_color = na, bgcolor = na, text_color = na))
            kz._hi_line.unshift(line.new(bar_index, high, bar_index, high, xloc = xloc.bar_index, style = line_style(session_style), color = c, width = session_width))
            kz._lo_line.unshift(line.new(bar_index, low, bar_index, low, xloc = xloc.bar_index, style = line_style(session_style), color = c, width = session_width))
            array.unshift(kz._hi_valid, true)
            array.unshift(kz._lo_valid, true)

            show_lbl = session_lbl != "Hide"
            kz._hi_label.unshift(label.new(time, high, show_lbl ? hi_txt : "", xloc = xloc.bar_time, color = colors_transparent, textcolor = c, style = label.style_label_down, size = text_size(session_lbl), text_font_family = font.family_monospace))
            kz._lo_label.unshift(label.new(time, low, show_lbl ? lo_txt : "", xloc = xloc.bar_time, color = colors_transparent, textcolor = c, style = label.style_label_up, size = text_size(session_lbl), text_font_family = font.family_monospace))
            del_kz(kz)

        adjust_in_kz(kz, t)
        adjust_out_kz(kz, t, t_prev)

selectedHTF = autoMode ? getHTFPeriod() : manualHTF
newPeriod = ta.change(time(selectedHTF)) != 0

[htfHasBear, htfHasBull, htfBearBot, htfBearTop, htfBullBot, htfBullTop, htfT1, htfT2] = request.security(syminfo.tickerid, selectedHTF, detectFVG(), lookahead = barmerge.lookahead_off)
htfCloseVal = request.security(syminfo.tickerid, selectedHTF, close, lookahead = barmerge.lookahead_off)

if newPeriod and not na(htfT1)
    string tfStr = formatTimeframe(selectedHTF)

    if showBearishFVG and htfHasBear and not na(htfBearBot) and not na(htfBearTop)
        bool exists = false
        if array.size(activeFVGs) > 0
            for i = 0 to array.size(activeFVGs) - 1
                item = array.get(activeFVGs, i)
                if not item.isBullish and item.top == htfBearTop and item.bottom == htfBearBot
                    exists := true
                    break

        if not exists
            box b = na
            line l = na
            label lbl = na
            midPrice = (htfBearTop + htfBearBot) / 2

            if showFVGBoxes
                b := box.new(htfT2, htfBearTop, time, htfBearBot, bgcolor = bearishFVGColor, border_color = color.new(color.gray, 100), xloc = xloc.bar_time)
                if showMidlines
                    l := line.new(htfT2, midPrice, time, midPrice, color = midlineColor, style = line_style(midlineStyle), width = midlineWidth, xloc = xloc.bar_time)

            if showLabels
                lbl := label.new(time, midPrice, "FVG-" + tfStr, color = color.new(color.white, 100), textcolor = labelTextColor, style = label.style_label_right, size = text_size(labelSize), xloc = xloc.bar_time, text_font_family = font.family_monospace)

            array.push(activeFVGs, FVG.new(b, l, lbl, htfBearTop, htfBearBot, false, htfT2, false, na))

    if showBullishFVG and htfHasBull and not na(htfBullBot) and not na(htfBullTop)
        bool exists = false
        if array.size(activeFVGs) > 0
            for i = 0 to array.size(activeFVGs) - 1
                item = array.get(activeFVGs, i)
                if item.isBullish and item.top == htfBullTop and item.bottom == htfBullBot
                    exists := true
                    break

        if not exists
            box b = na
            line l = na
            label lbl = na
            midPrice = (htfBullTop + htfBullBot) / 2

            if showFVGBoxes
                b := box.new(htfT2, htfBullTop, time, htfBullBot, bgcolor = bullishFVGColor, border_color = color.new(color.gray, 100), xloc = xloc.bar_time)
                if showMidlines
                    l := line.new(htfT2, midPrice, time, midPrice, color = midlineColor, style = line_style(midlineStyle), width = midlineWidth, xloc = xloc.bar_time)

            if showLabels
                lbl := label.new(time, midPrice, "FVG+" + tfStr, color = color.new(color.white, 100), textcolor = labelTextColor, style = label.style_label_right, size = text_size(labelSize), xloc = xloc.bar_time, text_font_family = font.family_monospace)

            array.push(activeFVGs, FVG.new(b, l, lbl, htfBullTop, htfBullBot, true, htfT2, false, na))

if newPeriod and array.size(activeFVGs) > 0
    for i = array.size(activeFVGs) - 1 to 0
        FVG f = array.get(activeFVGs, i)
        bool isInvalidated = f.isBullish ? (htfCloseVal < f.bottom) : (htfCloseVal > f.top)

        if isInvalidated
            if not na(f.fvgBox)
                box.delete(f.fvgBox)
            if not na(f.midline)
                line.delete(f.midline)
            if not na(f.fvgLabel)
                label.delete(f.fvgLabel)
            array.remove(activeFVGs, i)

if array.size(activeFVGs) > 0
    for i = 0 to array.size(activeFVGs) - 1
        FVG f = array.get(activeFVGs, i)
        if not na(f.fvgBox)
            box.set_bgcolor(f.fvgBox, filter_fvg ? (f.isBullish ? bullishFVGColor : bearishFVGColor) : color.new(color.white, 100))
            box.set_border_color(f.fvgBox, filter_fvg ? color.new(color.gray, 100) : color.new(color.white, 100))
            box.set_right(f.fvgBox, time)
        if not na(f.midline)
            line.set_color(f.midline, filter_fvg ? midlineColor : color.new(color.white, 100))
            line.set_x2(f.midline, time)
        if not na(f.fvgLabel)
            label.set_textcolor(f.fvgLabel, filter_fvg ? labelTextColor : color.new(color.white, 100))
            label.set_x(f.fvgLabel, time)

// Mark FVGs as mitigated the first time price trades back into them (partial fill),
// as distinct from full invalidation (handled above, which deletes the FVG outright).
if array.size(activeFVGs) > 0
    for fvg_item in activeFVGs
        if not fvg_item.mitigated and low <= fvg_item.top and high >= fvg_item.bottom
            fvg_item.mitigated := true
            fvg_item.mitigationTime := time

// "Max Mitigated FVGs to Keep": once an FVG has been touched (mitigated), it is a lower
// priority zone. Cap how many mitigated FVGs stay on the chart (oldest mitigated first);
// never prune untouched/active FVGs just because the total count is high.
mitigatedCount = 0
if array.size(activeFVGs) > 0
    for fvg_item in activeFVGs
        if fvg_item.mitigated
            mitigatedCount += 1

if mitigatedCount > maxFVGs
    excess = mitigatedCount - maxFVGs
    idx = 0
    while excess > 0 and idx < array.size(activeFVGs)
        FVG f = array.get(activeFVGs, idx)
        if f.mitigated
            if not na(f.fvgBox)
                box.delete(f.fvgBox)
            if not na(f.midline)
                line.delete(f.midline)
            if not na(f.fvgLabel)
                label.delete(f.fvgLabel)
            array.remove(activeFVGs, idx)
            excess -= 1
        else
            idx += 1

if filter_fvg and array.size(activeFVGs) > 0
    for fvg_item in activeFVGs
        if low <= fvg_item.top and high >= fvg_item.bottom
            if fvg_item.isBullish
                support_touch := true
            else
                resistance_touch := true

bool is_lower_tf = timeframe.isintraday and timeframe.in_seconds() <= 900

if showSessions and is_lower_tf and filter_session
    for [_, value] in key_zones
        manage_kz(value)

else if not filter_session and array.size(key_zones) > 0
    for [_, value] in key_zones
        kz k = value._kz
        while k._box.size() > 0
            box.delete(k._box.pop())
            line.delete(k._hi_line.pop())
            line.delete(k._lo_line.pop())
            label.delete(k._hi_label.pop())
            label.delete(k._lo_label.pop())
            array.pop(k._hi_valid)
            array.pop(k._lo_valid)

if filter_session and showSessions
    for [_, value] in key_zones
        kz k = value._kz
        if k.prevLowActive and low <= k.prevLow
            support_touch := true
            k.prevLowActive := false
        if k.prevHighActive and high >= k.prevHigh
            resistance_touch := true
            k.prevHighActive := false

if market_bias == 0 and bar_index > 10
    int has_cisd = close > open ? 1 : close < open ? -1 : 0
    if has_cisd == 0
        for k = 1 to 50
            has_cisd := close[k] > open[k] ? 1 : close[k] < open[k] ? -1 : 0
            if has_cisd != 0
                break
    if has_cisd != 0
        market_bias := has_cisd
        [origin_price, origin_time] = getCurrentLegExtreme(has_cisd)
        cisd_level := origin_price
        cisd_start_bar := origin_time
        swing_pivot := has_cisd == 1 ? high : low
        swing_pivot_bar := bar_index
        support_touch := false
        resistance_touch := false

int c_type = close > open ? 1 : close < open ? -1 : 0
float cur_price = na
int cur_timestamp = na

if market_bias == 1
    if high > swing_pivot
        swing_pivot := high
        swing_pivot_bar := bar_index
        if c_type == 1
            [p_price, p_idx] = getCurrentLegExtreme(1)
            cur_price := p_price
            cur_timestamp := p_idx
        else
            [p_price, p_idx] = getPreviousLegExtreme(1)
            cur_price := p_price
            cur_timestamp := p_idx
        cisd_level := cur_price
        cisd_start_bar := cur_timestamp

else if market_bias == -1
    if low < swing_pivot
        swing_pivot := low
        swing_pivot_bar := bar_index
        if c_type == -1
            [p_price, p_idx] = getCurrentLegExtreme(-1)
            cur_price := p_price
            cur_timestamp := p_idx
        else
            [p_price, p_idx] = getPreviousLegExtreme(-1)
            cur_price := p_price
            cur_timestamp := p_idx
        cisd_level := cur_price
        cisd_start_bar := cur_timestamp

bool bias_bullish = market_bias == -1 and close > cisd_level
bool bias_bearish = market_bias == 1 and close < cisd_level

bool any_filter_active = filter_session or filter_fvg
bool confluence_ok = not any_filter_active or (bias_bullish ? support_touch : bias_bearish ? resistance_touch : false)

if bias_bullish or bias_bearish
    bool draw_cisd = show_cisd and confluence_ok

    if draw_cisd
        color cisd_color = bias_bullish ? cisd_bull_color : cisd_bear_color

        string chosen_style = cisd_line_style
        line_st = chosen_style == "Solid" ? line.style_solid : chosen_style == "Dashed" ? line.style_dashed : line.style_dotted

        int cisd_line_x2 = bar_index + cisd_line_extend
        line cisd_line = line.new(cisd_start_bar, cisd_level, cisd_line_x2, cisd_level, color=cisd_color, width=cisd_line_width, style=line_st)

        label levelLabel = na
        if show_cisd_level
            // Pine labels have no native "bold" property, so bold Unicode glyphs
            // (Mathematical Sans-Bold) are used to render "CISD" visually bold.
            string cisd_bold_text = bias_bullish ? "𝗖𝗜𝗦𝗗 ▲" : "𝗖𝗜𝗦𝗗 ▼"
            levelLabel := label.new(
                 x = cisd_line_x2,
                 y = cisd_level,
                 text = cisd_bold_text,
                 style = label.style_label_left,
                 color = color.new(color.white, 100),
                 textcolor = cisd_color,
                 size = text_size(cisd_level_size))

        float runRange = math.abs(swing_pivot - cisd_level)
        float basePrice = swing_pivot
        bool bullish = bias_bullish
        int direction = bias_bullish ? 1 : -1
        label baseDot = na

        if show_projections
            string dot_style = bias_bullish ? label.style_label_up : label.style_label_down
            baseDot := label.new(
                x = swing_pivot_bar,
                y = basePrice,
                text = "●",
                style = dot_style,
                color = color.new(color.white, 100),
                textcolor = cisd_color,
                size = text_size(dot_size))

        array<label> projection_labels = array.new<label>()

        if show_projections
            string[] values = str.split(projection_levels, ",")
            for txt in values
                float mult = str.tonumber(str.trim(txt))
                if not na(mult)
                    float level_p = cisd_level + direction * mult * runRange
                    label lbl = label.new(
                         x = cisd_start_bar,
                         y = level_p,
                         text = str.tostring(mult) + " ●",
                         style = label.style_none,
                         color = color.new(color.white, 100),
                         textcolor = proj_color,
                         size = text_size(proj_size))
                    array.push(projection_labels, lbl)

        array.push(cisd_setups, cisdSetup.new(cisd_line, baseDot, levelLabel, projection_labels, basePrice, bullish))

    int new_emotional_state = bias_bullish ? 1 : -1
    market_bias := new_emotional_state
    [reset_price, reset_time] = getCurrentLegExtreme(new_emotional_state)
    cisd_level := reset_price
    cisd_start_bar := reset_time
    swing_pivot := new_emotional_state == 1 ? high : low
    swing_pivot_bar := bar_index
    support_touch := false
    resistance_touch := false

while array.size(cisd_setups) > cisd_maxLines
    cisdSetup oldest = array.shift(cisd_setups)
    if not na(oldest.cisdLine)
        line.delete(oldest.cisdLine)
    if not na(oldest.baseDot)
        label.delete(oldest.baseDot)
    if not na(oldest.levelLabel)
        label.delete(oldest.levelLabel)
    for lb in oldest.projection_labels
        label.delete(lb)

int j_idx = array.size(cisd_setups) - 1
while j_idx >= 0
    cisdSetup setup = array.get(cisd_setups, j_idx)
    bool touched = setup.bullish ? (low <= setup.basePrice) : (high >= setup.basePrice)

    if touched
        if not na(setup.cisdLine)
            line.delete(setup.cisdLine)
        if not na(setup.baseDot)
            label.delete(setup.baseDot)
        if not na(setup.levelLabel)
            label.delete(setup.levelLabel)
        for lb in setup.projection_labels
            label.delete(lb)
        array.remove(cisd_setups, j_idx)
    j_idx -= 1

var table topTable = na
if barstate.islast and show_table
    final_pos = get_position(table_pos_str)
    ticker_sz = get_size(ticker_size_str)
    label_sz = get_size(label_size_str)

    string auto_date = str.format("{0,date,EEEE, MMM dd, yyyy}", time)

    if na(topTable)
        topTable := table.new(final_pos, 1, 3, bgcolor = bg_col, frame_width = frame_w, frame_color = border_col, border_width = border_w, border_color = border_col)

    table.cell(topTable, 0, 0, auto_date, text_color = text_col, text_size = label_sz, text_halign = text.align_center)
    table.cell(topTable, 0, 1, "CISD PRO+", text_color = text_col, text_size = ticker_sz, text_halign = text.align_center)
    table.cell(topTable, 0, 2, syminfo.ticker + " • " + to_friendly_tf(timeframe.period), text_color = text_col, text_size = label_sz, text_halign = text.align_center)



