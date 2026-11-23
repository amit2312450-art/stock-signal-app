import streamlit as st
import yfinance as yf
import pandas as pd

st.set_page_config(page_title="My Buy/Sell Signal App", layout="centered")
st.title("📈 Stock Buy / Sell Signal Finder")

ticker = st.text_input("Stock Symbol daalein (NSE ke liye .NS lagayein):", "IRFC.NS")

def to_float(val):
    if hasattr(val, "iloc"):
        val = val.iloc[-1]
    if hasattr(val, "item"):
        return float(val.item())
    return float(val)

if st.button("Signal Check Karein"):
    try:
        data = yf.download(ticker.strip(), period="6mo", interval="1d", progress=False)

        if data is not None and not data.empty and len(data) >= 50:
            close = data["Close"]
            if isinstance(close, pd.DataFrame):
                close = close.iloc[:, 0]

            delta = close.diff()
            gain = delta.where(delta > 0, 0).rolling(window=14).mean()
            loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
            rs = gain / loss
            rsi = 100 - (100 / (1 + rs))

            ema_20 = close.ewm(span=20, adjust=False).mean()
            ema_50 = close.ewm(span=50, adjust=False).mean()

            latest_close = to_float(close)
            latest_rsi = to_float(rsi)
            ema20_val = to_float(ema_20)
            ema50_val = to_float(ema_50)

            st.metric(label="Current Price (₹)", value=f"₹{latest_close:.2f}")
            st.write(f"**RSI (14):** {latest_rsi:.2f}")
            st.write(f"**EMA 20:** ₹{ema20_val:.2f} | **EMA 50:** ₹{ema50_val:.2f}")

            if latest_rsi < 35 and ema20_val > ema50_val:
                st.success("🟢 STRONG BUY SIGNAL (Oversold + Uptrend)")
            elif latest_rsi < 30:
                st.success("🟢 BUY SIGNAL (Stock Oversold hai)")
            elif latest_rsi > 70:
                st.error("🔴 SELL SIGNAL (Stock Overbought hai)")
            elif ema20_val < ema50_val and latest_rsi > 55:
                st.warning("🟠 SELL / AVOID (Downtrend shuru ho sakta hai)")
            else:
                st.info("🟡 HOLD / WAIT (Abhi koi clear signal nahi hai)")

            chart_df = pd.DataFrame({
                "Close": close,
                "EMA_20": ema_20,
                "EMA_50": ema_50
            })
            st.line_chart(chart_df)
        else:
            st.error("Data nahi mila ya history kam hai. Symbol check karein (jaise: NHPC.NS, IRFC.NS, TATAGOLD.NS).")
    except Exception as e:
        st.error(f"Kuch error aaya: {e}")
