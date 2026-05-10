---
author: "Kyle Jones"
date_published: "August 11, 2025"
date_exported_from_medium: "November 10, 2025"
canonical_link: "https://medium.com/@kyle-t-jones/visualizing-wti-oil-prices-with-a-mountain-plot-and-threshold-fill-in-python-e213a7391498"
---

# Visualizing WTI Oil Prices with a "Mountain Plot" and Threshold Fill in Python

Most oil price charts are a single line which can be useful, but lacks context. In the oil market, specific price levels carry operational...

### Visualizing WTI Oil Prices with a "Mountain Plot" and Threshold Fill in Python 

Most oil price charts are a single line which can be useful, but lacks context. In the oil market, specific price levels carry operational meaning --- especially for producers.

In this project, I wanted to visualize WTI crude oil prices in a way that tells more of the story. The result is a "mountain plot" that fills the area above and below meaningful thresholds in different colors. It animates over time so you can watch the market move through cycles of profitability and pressure.

### Why These Two Price Levels Matter
I chose two reference lines for this visualization. **\$70/bbl (solid line)** is often cited as the break-even point for US shale production. When WTI is above this level, many shale projects are profitable, and drilling tends to accelerate. \$50/bbl (dashed line) is a rough global average break-even for conventional production. Below this point, pressure is felt industry-wide.

By shading green above \$70 and red below, the plot instantly shows when prices are in "comfort" or "stress" territory for US shale.

### The Finished Chart
The animation below uses WTI data from Yahoo Finance (`CL=F`), downsampled to keep file size small and performance fast.

You can see the long stretches in the red zone during the 2014--2016 price collapse and the 2020 pandemic shock, as well as the extended green period since the post-2021 rebound.


### The Code
This notebook-friendly Python code will fetch WTI data from Yahoo Finance, create the mountain plot with Matplotlib, and stream frames to disk to avoid memory problems.

```python
# %pip install yfinance imageio --quiet

import io, math
import numpy as np
import pandas as pd
import matplotlib as mpl
import matplotlib.pyplot as plt
from matplotlib.dates import AutoDateLocator, ConciseDateFormatter
import imageio.v2 as imageio
import yfinance as yf

# --- Config ---
start_date = "2010-01-01"
end_date = None
out_gif = "wti_mountain.gif"
out_png = "wti_mountain.png"
title = "WTI Crude Oil Price"
fps = 24
dpi = 160
y_shale = 70.0
y_global = 50.0
max_frames = 600

# --- Style ---
mpl.rcParams.update({
    'axes.grid': False,
    "font.family": "serif",
    "axes.spines.top": False,
    "axes.spines.right": False,
    "axes.linewidth": 1.1,
    "axes.labelsize": 11,
    "axes.titlesize": 13,
    "xtick.labelsize": 9,
    "ytick.labelsize": 9,
    "figure.dpi": 120,
    "savefig.bbox": "tight",
    "savefig.pad_inches": 0.08,
})
def _bracket(ax):
    ax.spines["left"].set_position(("outward", 6))
    ax.spines["bottom"].set_position(("outward", 6))

# --- Fetch data ---
t = yf.Ticker("CL=F")
df = t.history(start=start_date, end=end_date, auto_adjust=True).reset_index()
df = df[["Date","Close"]].rename(columns={"Date":"date","Close":"price"}).dropna()
df["date"] = pd.to_datetime(df["date"])
df = df.set_index("date").resample("W-FRI").last().dropna().reset_index()
# Downsample for performance
stride = max(1, math.ceil(len(df) / max_frames))
df_anim = df.iloc[::stride]
dates = df_anim["date"].to_numpy()
prices = df_anim["price"].to_numpy(float)
y_min = math.floor(min(prices.min(), y_global) / 5) * 5
y_max = math.ceil(max(prices.max(), y_shale) / 5) * 5
locator = AutoDateLocator()
fig, ax = plt.subplots(figsize=(10, 4.5))
def draw_frame(k):
    ax.clear()
    _bracket(ax)
    ax.set_title(title)
    ax.set_ylabel("USD per barrel")
    ax.set_ylim(y_min, y_max)
    ax.xaxis.set_major_locator(locator)
    ax.xaxis.set_major_formatter(ConciseDateFormatter(locator))
    x = dates[:k+1]
    y = prices[:k+1]
    ax.fill_between(x, y_shale, y, where=y >= y_shale, interpolate=True, alpha=0.55, color="#2ca02c")
    ax.fill_between(x, y, y_shale, where=y < y_shale, interpolate=True, alpha=0.55, color="#d62728")
    ax.axhline(y_shale, lw=1.1, color="0.35", alpha=0.9)
    ax.axhline(y_global, lw=1.0, color="0.55", ls="--", alpha=0.8)
    if len(x):
        ax.text(x[0], y_shale+1.2, "$70  US shale price", va="bottom", ha="left", fontsize=9, color="0.2")
        ax.text(x[0], y_global+1.2, "$50  Global average", va="bottom", ha="left", fontsize=9, color="0.3")
    ax.plot(x, y, lw=1.2, color="black", alpha=0.95)

# Save final frame
draw_frame(len(dates)-1)
plt.savefig(out_png, dpi=dpi)
plt.show()

# Stream GIF to disk
with imageio.get_writer(out_gif, mode="I", fps=fps) as w:
    for k in range(len(dates)):
        draw_frame(k)
        buf = io.BytesIO()
        fig.savefig(buf, format="png", dpi=dpi)
        buf.seek(0)
        w.append_data(imageio.imread(buf))
        buf.close()
plt.close(fig)
print(f"Wrote {out_png} and {out_gif}")
```

### What This Adds to the Analysis
Beyond aesthetics, this type of plot gives decision-makers a faster way to connect price history with operational thresholds. It becomes obvious when the market was in "comfort" vs. "stress" zones and for how long. That's useful for producers gauging when to accelerate or slow drilling, investors assessing cyclicality and capital discipline, and analysts framing discussions of supply response to price changes. If you try this code, experiment with different thresholds for other commodities, or even for entirely different markets --- such as electricity, gas, or metals --- where profitability zones matter.
