# Python-dashboard-g
import sys
import subprocess
import os
import time

# --- AUTO-INSTALLATION BLOCK ---
def install_package(package):
    print(f"Package '{package}' not found. Installing now... (This may take a minute)")
    try:
        subprocess.check_call([sys.executable, "-m", "pip", "install", package])
        print(f"Successfully installed {package}.")
    except subprocess.CalledProcessError:
        print(f"Failed to install {package}. Please install it manually using 'pip install {package}'.")

required_libraries = ['pandas', 'plotly', 'dash', 'requests']

print("Checking dependencies...")
for lib in required_libraries:
    try:
        __import__(lib)
    except ImportError:
        install_package(lib)

# --- IMPORTS ---
import pandas as pd
import plotly.express as px
from dash import Dash, dcc, html
from dash.dependencies import Input, Output
import requests
import io

# --- 1. Data Loading and Preprocessing ---

SHEET_ID = '111ltSPJCAv2kxb9keSPGD83ByhPZwCWKRjD4ZxMAOFQ'
GID = '406151901'
google_sheet_url = f'https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&gid={GID}'

try:
    print("Step 1/3: Fetching data from Google Sheets (approx 10MB)...")
    start_time = time.time()
    response = requests.get(google_sheet_url)
    response.raise_for_status()

    # Read CSV
    data = response.content.decode('utf-8')
    df = pd.read_csv(io.StringIO(data))
    print(f"   - Download complete in {time.time() - start_time:.2f} seconds.")

    if df.empty or 'Ratings' not in df.columns or 'Review Date' not in df.columns:
        print("Error: DataFrame is empty or missing columns.")
        df = None
    else:
        print("Step 2/3: Processing 200k+ rows...")
        # OPTIMIZATION: Specify format explicitly to speed up parsing by ~10x
        df['Review Date'] = pd.to_datetime(df['Review Date'], format='%Y-%m-%d %H:%M:%S', errors='coerce')

        df.dropna(subset=['Review'], inplace=True)

        overall_average_rating = df['Ratings'].mean()

        df['Review Month'] = df['Review Date'].dt.to_period('M')
        monthly_ratings = df.groupby('Review Month')['Ratings'].mean().reset_index()
        monthly_ratings['Review Month'] = monthly_ratings['Review Month'].astype(str)

        ratings_count = df['Ratings'].value_counts().sort_index().reset_index()
        ratings_count.columns = ['Rating', 'Count']
        print("   - Processing complete.")

except Exception as e:
    print(f"Error: {e}")
    df = None

# --- 2. Initialize App ---
if df is None:
    app = Dash(__name__)
    app.layout = html.Div([html.H1("Error Loading Data", style={'color': 'red'})])
else:
    print("Step 3/3: Starting Dashboard Server...")
    app = Dash(__name__)
    server = app.server

    colors = {
        'background': '#F9F9F9',
        'text': '#333333',
        'accent': '#1F77B4',
        'card_bg': '#FFFFFF'
    }

    app.layout = html.Div(style={'backgroundColor': colors['background'], 'padding': '20px'}, children=[
        html.H1('ChatGPT Reviews Dashboard', style={'textAlign': 'center', 'color': colors['text']}),
        html.Hr(style={'borderTop': '1px solid #ddd'}),

        html.Div(style={'display': 'flex', 'justifyContent': 'center', 'marginBottom': '20px'}, children=[
            html.Div(style={'backgroundColor': colors['card_bg'], 'padding': '15px', 'borderRadius': '5px', 'boxShadow': '2px 2px 2px lightgrey', 'textAlign': 'center', 'width': '300px'}, children=[
                html.H3('Overall Average Rating', style={'color': colors['text'], 'marginBottom': '5px'}),
                html.P(f'{overall_average_rating:.2f} / 5.0', style={'fontSize': '2.5em', 'fontWeight': 'bold', 'color': colors['accent']})
            ])
        ]),

        html.Div(className='row', style={'display': 'flex', 'flexDirection': 'row', 'flexWrap': 'wrap', 'justifyContent': 'space-between'}, children=[
            html.Div(style={'width': '48%', 'minWidth': '300px', 'marginBottom': '20px', 'backgroundColor': colors['card_bg'], 'padding': '15px', 'borderRadius': '5px', 'boxShadow': '2px 2px 2px lightgrey'}, children=[
                html.H3('Ratings Distribution', style={'textAlign': 'center', 'color': colors['text']}),
                dcc.Graph(figure=px.bar(ratings_count, x='Rating', y='Count', color='Rating', title='Count of Reviews', labels={'Rating': 'Score', 'Count': 'Reviews'}, color_continuous_scale=px.colors.sequential.Plasma, template='plotly_white').update_layout(xaxis={'tickmode': 'linear', 'dtick': 1}))
            ]),
            html.Div(style={'width': '48%', 'minWidth': '300px', 'marginBottom': '20px', 'backgroundColor': colors['card_bg'], 'padding': '15px', 'borderRadius': '5px', 'boxShadow': '2px 2px 2px lightgrey'}, children=[
                html.H3('Monthly Trend', style={'textAlign': 'center', 'color': colors['text']}),
                dcc.Graph(figure=px.line(monthly_ratings, x='Review Month', y='Ratings', title='Average Rating Trend', markers=True, template='plotly_white').update_traces(line_color=colors['accent']).update_layout(yaxis_range=[3, 5]))
            ])
        ])
    ])

if __name__ == '__main__':
    if df is not None:
        print("Done! Dashboard running at http://127.0.0.1:8050/")
        app.run(debug=True)
