
# Import necessary libraries
import streamlit as st
import pandas as pd
import plotly.express as px
import numpy as np
import os
import glob

# --- Streamlit App Configuration ---
st.set_page_config(layout="wide")
st.title('📊 CSV Score Distribution Viewer')
st.write("This dashboard visualizes a histogram of scores from a CSV file. Adjust controls in the sidebar.")

# --- Sidebar Configuration ---
st.sidebar.header("⚙️ Controls")

# Try to automatically find a CSV file in the same directory
csv_files = glob.glob(os.path.join(os.path.dirname(__file__), "*.csv"))
uploaded_file = None

if csv_files:
    auto_file_path = csv_files[0]
    st.sidebar.markdown(f"📂 Automatically loading: `{os.path.basename(auto_file_path)}`")
    uploaded_file = open(auto_file_path, "rb")
else:
    uploaded_file = st.sidebar.file_uploader("1. Choose a CSV file", type="csv")

# Initialize
df = None
min_score = None
max_score = None

# --- Load and Clean Data ---
if uploaded_file is not None:
    try:
        df = pd.read_csv(uploaded_file)

        if 'Score' not in df.columns:
            st.error("❌ Error: The uploaded CSV file must contain a column named 'Score'.")
            st.stop()

        df['Score'] = pd.to_numeric(df['Score'], errors='coerce')
        original_count = len(df)
        df.dropna(subset=['Score'], inplace=True)
        cleaned_count = len(df)

        if cleaned_count < original_count:
            st.warning(f"⚠️ Removed {original_count - cleaned_count} rows with non-numeric 'Score' values.")

        if df.empty:
            st.error("❌ Error: No valid numeric 'Score' data found in the uploaded file after cleaning.")
            st.stop()

        min_score = df['Score'].min()
        max_score = df['Score'].max()

    except Exception as e:
        st.error(f"❌ Error while processing the file: {e}")
        st.stop()

# --- Main Display Controls ---
if df is not None and min_score is not None and max_score is not None:
    default_threshold = min(10.0, float(max_score) + 0.001)

    # Number input for threshold trimming
    score_threshold = st.sidebar.number_input(
        label='2. Trim scores greater than:',
        min_value=float(min_score),
        max_value=float(max_score) + 0.001,
        value=default_threshold,
        step=0.1,
        format="%.4f",
        help="Exclude scores above this value from the histogram."
    )

    # Sidebar input for number of histogram bins
    suggested_bins = min(100, max(10, int((max_score - min_score) / 0.1)))
    num_bins = st.sidebar.number_input(
        label='3. Number of Histogram Bins:',
        min_value=1,
        max_value=500,
        value=suggested_bins,
        step=1,
        help="Choose the number of bins for the histogram."
    )

    # Filter data
    filtered_df = df[df['Score'] <= score_threshold].copy()

    # --- Display Output ---
    st.header("Histogram of Scores")

    if filtered_df.empty:
        st.warning(f"⚠️ No data points remaining after filtering scores below {score_threshold:.4f}.")
    else:
        st.write(f"Displaying histogram for scores ≤ **{score_threshold:.4f}**.")
        st.write(f"**{len(filtered_df)}** data points shown (out of **{len(df)}** total).")

        fig = px.histogram(
            filtered_df,
            x='Score',
            nbins=num_bins,
            title=f'Distribution of Scores (Up to {score_threshold:.4f}, {num_bins} Bins)',
            labels={'Score': 'Score Value', 'count': 'Frequency'},
            opacity=0.8,
            template='plotly_white'
        )

        fig.update_layout(
            bargap=0.1,
            xaxis_title='Score',
            yaxis_title='Number of Entries'
        )

        st.plotly_chart(fig, use_container_width=True)

        if st.checkbox("📋 Show Filtered Data Table"):
            cols = ['Score'] + [col for col in filtered_df.columns if col != 'Score']
            st.dataframe(filtered_df[cols])
else:
    st.info("👈 Upload a CSV file or place one in this directory to get started.")
