import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import json
import os
from datetime import datetime
from pycoingecko import CoinGeckoAPI

# Initialize
DATA_FILE = "assets_data.json"
UPLOAD_DIR = "uploaded_sources"
os.makedirs(UPLOAD_DIR, exist_ok=True)

cg = CoinGeckoAPI()

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    return {"assets": {}}

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)

data = load_data()

st.set_page_config(page_title="My Asset Thesis Dashboard", layout="wide")
st.title("🧠 My Stocks & Cryptos Dashboard")

# Sidebar
st.sidebar.header("Manage Assets")
new_ticker = st.sidebar.text_input("Add Asset (e.g., AAPL or BTC)", "").upper().strip()
asset_type = st.sidebar.selectbox("Type", ["Stock", "Crypto"])

if st.sidebar.button("Add Asset"):
    if new_ticker and new_ticker not in data["assets"]:
        data["assets"][new_ticker] = {
            "type": asset_type,
            "thesis": "",
            "sources": [],
            "added": datetime.now().isoformat()
        }
        save_data(data)
        st.success(f"Added {new_ticker}")

# Main App
if not data["assets"]:
    st.info("Add assets from the sidebar to get started.")
else:
    ticker = st.selectbox("Select Asset", list(data["assets"].keys()))
    asset = data["assets"][ticker]
    
    st.subheader(f"{ticker} — {asset['type']}")

    # ==================== CHARTS SECTION ====================
    st.subheader("📊 Price Charts")
    
    # Timeframe selector
    timeframe = st.selectbox(
        "Select Timeframe",
        ["1d", "5d", "1mo", "3mo", "6mo", "1y", "2y", "5y", "max"],
        index=2
    )
    
    chart_type = st.radio("Chart Type", ["Candlestick + Indicators", "Line Chart"], horizontal=True)
    
    col1, col2 = st.columns([3, 1])
    
    with col1:
        try:
            if asset["type"] == "Stock":
                stock = yf.Ticker(ticker)
                period_map = {"1d": "1d", "5d": "5d", "1mo": "1mo", "3mo": "3mo", 
                             "6mo": "6mo", "1y": "1y", "2y": "2y", "5y": "5y", "max": "max"}
                
                hist = stock.history(period=period_map[timeframe], interval="1d" if timeframe != "1d" else "5m")
                
                if not hist.empty:
                    # Main Chart
                    if chart_type == "Candlestick + Indicators":
                        fig = make_subplots(rows=3, cols=1, shared_xaxes=True, 
                                          row_heights=[0.6, 0.2, 0.2], vertical_spacing=0.05)
                        
                        # Candlestick
                        fig.add_trace(go.Candlestick(x=hist.index,
                                                    open=hist['Open'], high=hist['High'],
                                                    low=hist['Low'], close=hist['Close'],
                                                    name="Price"), row=1, col=1)
                        
                        # Moving Averages
                        hist['SMA50'] = hist['Close'].rolling(50).mean()
                        hist['SMA200'] = hist['Close'].rolling(200).mean()
                        
                        fig.add_trace(go.Scatter(x=hist.index, y=hist['SMA50'], name="SMA50", line=dict(color='orange')), row=1, col=1)
                        fig.add_trace(go.Scatter(x=hist.index, y=hist['SMA200'], name="SMA200", line=dict(color='red')), row=1, col=1)
                        
                        # Volume
                        fig.add_trace(go.Bar(x=hist.index, y=hist['Volume'], name="Volume", marker_color='rgba(0, 150, 255, 0.6)'), row=2, col=1)
                        
                        # RSI
                        delta = hist['Close'].diff()
                        gain = (delta.where(delta > 0, 0)).rolling(14).mean()
                        loss = (-delta.where(delta < 0, 0)).rolling(14).mean()
                        rs = gain / loss
                        rsi = 100 - (100 / (1 + rs))
                        
                        fig.add_trace(go.Scatter(x=hist.index, y=rsi, name="RSI(14)", line=dict(color='purple')), row=3, col=1)
                        fig.add_hline(y=70, line_dash="dash", line_color="red", row=3, col=1)
                        fig.add_hline(y=30, line_dash="dash", line_color="green", row=3, col=1)
                        
                        fig.update_layout(height=700, title=f"{ticker} - Candlestick with Indicators")
                        fig.update_xaxes(rangeslider_visible=False)
                        
                    else:  # Line Chart
                        fig = go.Figure()
                        fig.add_trace(go.Scatter(x=hist.index, y=hist['Close'], name="Close Price", line=dict(width=2.5)))
                        fig.update_layout(title=f"{ticker} Price History", height=500)
                    
                    st.plotly_chart(fig, use_container_width=True)
                    
                    # Current Price Metric
                    current_price = hist['Close'][-1]
                    prev_price = hist['Close'][-2] if len(hist) > 1 else current_price
                    change = ((current_price - prev_price) / prev_price) * 100
                    st.metric("Latest Price", f"${current_price:.2f}", f"{change:+.2f}%")
                    
            else:  # Crypto
                # Crypto handling with CoinGecko + fallback
                coins = cg.get_coins_list()
                coin_id = next((coin['id'] for coin in coins if coin['symbol'].upper() == ticker), None)
                
                if coin_id:
                    days_map = {"1d": 1, "5d": 5, "1mo": 30, "3mo": 90, "6mo": 180, "1y": 365, "2y": 730, "5y": 1825, "max": "max"}
                    days = days_map[timeframe]
                    
                    market_chart = cg.get_coin_market_chart_by_id(coin_id, vs_currency='usd', days=days)
                    df = pd.DataFrame(market_chart['prices'], columns=['timestamp', 'price'])
                    df['date'] = pd.to_datetime(df['timestamp'], unit='ms')
                    
                    vol_df = pd.DataFrame(market_chart['total_volumes'], columns=['timestamp', 'volume'])
                    vol_df['date'] = pd.to_datetime(vol_df['timestamp'], unit='ms')
                    
                    # Plot
                    fig = make_subplots(rows=2, cols=1, shared_xaxes=True, row_heights=[0.7, 0.3])
                    
                    fig.add_trace(go.Scatter(x=df['date'], y=df['price'], name="Price", line=dict(width=3)), row=1, col=1)
                    fig.add_trace(go.Bar(x=vol_df['date'], y=vol_df['volume'], name="Volume", marker_color='rgba(100, 150, 255, 0.6)'), row=2, col=1)
                    
                    fig.update_layout(height=600, title=f"{ticker} Price & Volume")
                    st.plotly_chart(fig, use_container_width=True)
                    
                    price_now = df['price'].iloc[-1]
                    st.metric("Current Price", f"${price_now:,.4f}")
                    
                else:
                    st.error("Crypto data not available")
                    
        except Exception as e:
            st.error(f"Error loading chart: {e}")

    # ==================== Thesis & Sources (unchanged) ====================
    st.subheader("📝 My Thesis / Opinion")
    thesis = st.text_area("Write your analysis, thesis, risks, catalysts, etc.", 
                         asset.get("thesis", ""), height=250)
    if st.button("💾 Save Thesis"):
        data["assets"][ticker]["thesis"] = thesis
        save_data(data)
        st.success("Thesis saved!")

    st.subheader("🔗 Sources & Macro Influences")
    # ... (same sources section as before)
    src_col1, src_col2 = st.columns(2)
    with src_col1:
        link_name = st.text_input("Source Name/Title")
        link_url = st.text_input("URL/Link")
    with src_col2:
        uploaded_file = st.file_uploader("Upload supporting file", type=["pdf", "png", "jpg", "jpeg", "csv", "txt"])
    
    if st.button("Add Source"):
        new_src = {"name": link_name or "Untitled", "url": link_url, "date": datetime.now().isoformat()}
        if uploaded_file:
            file_path = os.path.join(UPLOAD_DIR, f"{ticker}_{datetime.now().strftime('%Y%m%d_%H%M')}_{uploaded_file.name}")
            with open(file_path, "wb") as f:
                f.write(uploaded_file.getbuffer())
            new_src["file_path"] = file_path
        asset["sources"].append(new_src)
        save_data(data)
        st.rerun()

    # Display sources (same as previous version)
    if asset.get("sources"):
        for i, src in enumerate(reversed(asset["sources"])):
            with st.expander(f"{src['name']} — {src.get('date', '')[:10]}"):
                if src.get("url"):
                    st.markdown(f"[🔗 Open Link]({src['url']})")
                if src.get("file_path") and os.path.exists(src["file_path"]):
                    st.download_button("📥 Download File", 
                                     data=open(src["file_path"], "rb").read(),
                                     file_name=os.path.basename(src["file_path"]),
                                     key=f"dl_{i}")
    else:
        st.caption("No sources added yet.")

st.sidebar.success("Enhanced charts with candlesticks, MA, RSI & volume added!")
