import streamlit as st
import yfinance as yf
import pandas as pd
import numpy as np
import plotly.graph_objects as go
from datetime import datetime

st.set_page_config(page_title="Trading Pro Mobile", layout="wide")

st.title("🚀 Trading Pro - SMC + APA")
st.markdown("**Smart Money Concepts + Advanced Price Action**")

# Sidebar
st.sidebar.header("Settings")
symbol = st.sidebar.text_input("Symbol (e.g. AAPL, BTC-USD)", "AAPL")
period = st.sidebar.selectbox("Period", ["1mo", "3mo", "6mo", "1y"], index=3)

def fetch_data(symbol, period):
    try:
        df = yf.download(symbol, period=period, progress=False)
        return df
    except:
        st.error("Could not fetch data. Try again later.")
        return pd.DataFrame()

def calculate_indicators(df):
    df = df.copy()
    df['SMA_50'] = df['Close'].rolling(50).mean()
    df['SMA_200'] = df['Close'].rolling(200).mean()
    delta = df['Close'].diff()
    gain = delta.where(delta > 0, 0).rolling(14).mean()
    loss = -delta.where(delta < 0, 0).rolling(14).mean()
    df['RSI'] = 100 - (100 / (1 + rs)) if (rs := gain / loss) > 0 else 50
    return df

def smc_apa_analysis(df):
    df = df.copy()
    # Simple SMC
    df['FVG_Bull'] = np.where(df['Low'].shift(1) > df['High'], (df['Low'].shift(1) + df['High'])/2, np.nan)
    trend = "Bullish" if df['SMA_50'].iloc[-1] > df['SMA_200'].iloc[-1] else "Bearish"
    
    latest = df.iloc[-1]
    if trend == "Bullish" and not pd.isna(latest.get('FVG_Bull', np.nan)):
        signal = "Strong Buy" if latest['RSI'] < 45 else "Buy"
    elif trend == "Bearish":
        signal = "Strong Sell" if latest['RSI'] > 55 else "Sell"
    else:
        signal = "Hold"
    
    return signal, trend, df

if st.sidebar.button("🔥 Analyze", type="primary"):
    with st.spinner("Fetching market data..."):
        df = fetch_data(symbol, period)
        if df.empty:
            st.stop()
        
        df = calculate_indicators(df)
        signal, trend, df = smc_apa_analysis(df)
        latest = df.iloc[-1]
        
        # Mobile-friendly metrics
        col1, col2 = st.columns(2)
        with col1:
            st.metric("Price", f"${latest['Close']:.2f}")
            st.metric("Signal", signal)
        with col2:
            st.metric("RSI", f"{latest['RSI']:.1f}")
            st.metric("Trend", trend)
        
        # Chart (optimized for phone)
        fig = go.Figure()
        fig.add_trace(go.Candlestick(x=df.index, open=df['Open'], high=df['High'],
                                    low=df['Low'], close=df['Close'], name="Price"))
        fig.add_trace(go.Scatter(x=df.index, y=df['SMA_50'], name="SMA 50", line=dict(color='orange')))
        fig.add_trace(go.Scatter(x=df.index, y=df['SMA_200'], name="SMA 200", line=dict(color='blue')))
        
        fig.update_layout(height=500, template="plotly_dark", 
                         title=f"{symbol} - SMC + APA", 
                         margin=dict(l=10, r=10, t=50, b=10))
        st.plotly_chart(fig, use_container_width=True)
        
        st.subheader("Last 10 Candles")
        st.dataframe(df.tail(10)[['Close', 'RSI']].round(2), use_container_width=True)

# Strategy Guide (collapsed for mobile)
with st.expander("📖 SMC + APA Strategy Guide"):
    st.markdown("""
    - **Analysis**: Check trend with SMA50/200
    - **POI**: Look for Fair Value Gaps (green zones)
    - **Action**: Trade in direction of trend with RSI confirmation
    """)