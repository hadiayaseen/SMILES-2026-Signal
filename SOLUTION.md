##### Reproducibility Instructions

###### Required environment

* need to run the below: !pip install numpy scipy gdown
* no GPU available in my pc
* Tested on Windows with Anaconda

###### How to run

i used  Jupyter notebook to run applicant\_solution.ipynb

This single command will:

1. Automatically download `challenge.mat` from Google Drive
2. Run the baseline for comparison
3. Run the full canceller
4. Write `results.json`

Expected runtime: 2–5 minutes on a standard laptop.

###### Expected output


=== Baseline ===
  ch0: 3.98 dB
  ch1: 4.86 dB
  ch2: 3.49 dB
  ch3: 3.74 dB
  Metric \[baseline]: 4.02 dB

=== Your Solution ===
  ch0: 9.41 dB
  ch1: 7.15 dB
  ch2: 8.63 dB
  ch3: 7.29 dB
  Metric \[yours]: 8.12 dB

Baseline : 4.02 dB
Yours    : 8.12 dB
Gain     : +4.10 dB


##### Final Solution Description

###### What was modified

Only `applicant\_solution.py` was modified. Two things were added to it:

1. A new function `rank1\_cancel()` that removes the external spatial interference
2. A new `your\_canceller()` function that replaces the placeholder

##### 

##### Understanding the problem

The received signal on each channel is:
RX\[n, c] = desired\[n, c] + F\_c(TX) + E\[n, c] + noise


There are two interference components to remove:

* `F\_c(TX)` — nonlinear distortion caused by the device's own transmission.
This produces IM3 (3rd-order intermodulation) cross-products that fall
inside the receive band. Example: transmitting on frequencies A and B
produces a spur at 2A − B inside the RX band.
* `E\[n, c]` — an external jammer. The same waveform appears on all 4 RX
channels simultaneously, just with different complex gains per channel.
This is a rank-1 spatial structure.



##### Final approach

###### rank-1 cancellation first, then TX cancellation

The final pipeline related to RX is:


**Step 1 — Rank-1 external spatial cancellation**

The external jammer satisfies:
RX\[:, c] ≈ alpha\_c · E\[n]   for c = 0, 1, 2, 3


This is a rank-1 matrix — one shared waveform, four different complex gains.
To extract and remove it:

1. Band-pass filter all 4 RX channels into the interference band
2. Compute the 4×4 spatial covariance matrix: `R = X^H · X / N`
3. Find the dominant eigenvector via `numpy.linalg.eigh` — this is the
direction of maximum shared power across all 4 channels
4. Project onto this direction to recover the shared waveform `E\[n]`
5. For each channel c, compute the complex gain:
`alpha\_c = <E, band\_c> / ||E||²`

Subtract: `RX\_clean\[:, c] = RX\[:, c] − alpha\_c · E\[n]`

This method removes the interference by making the system effectively ignore signals coming from the jammer’s direction.



**Step 2 — TX nonlinear cancellation**

After the jammer is removed, the residual still contains IM3 distortion from the device's own transmission. This is handled by the baseline method:
fit a linear model over 10 IM3 cross-product terms at 13 time lags (±6 samples) using least squares, then subtract the prediction.

Mathematically this is channel estimation:
H = (X^H X)^{-1} X^H y


where X is the matrix of IM3 features and y is the band-filtered RX channel.



###### Why this order matters 



When TX cancellation is done first (as in the baseline), the IM3 regression is fitting against a target that still contains the external jammer. The jammer adds structured noise to the regression target, which biases the coefficient estimates and limits cancellation depth.

By removing the external jammer first, the TX regression sees a much cleaner target signal. The least-squares fit converges to better coefficients and removes significantly more IM3 interference.

|Order|Score|
|-|-|
|TX first → rank-1 second|7.01 dB|
|Rank-1 first → TX second|**8.12 dB**|

This single change — reversing the order — gave +1.11 dB improvement.

### 

###### What contributed most to the final score

|Stage|Contribution|
|-|-|
|Baseline TX cancellation alone|4.02 dB|
|Adding rank-1 cancellation (TX first)|7.01 dB (+2.99 dB)|
|Reversing order (rank-1 first)|**8.12 dB (+1.11 dB)**|

The rank-1 spatial cancellation step contributed the most (+2.99 dB), and the order reversal provided an additional significant gain (+1.11 dB).



##### Experiments and Failed Attempts

###### Attempt 1: Full iteration — TX → rank-1 → TX → rank-1 (twice)

I tried **to r**un the full two-stage pipeline twice in a loop, re-fitting both stages on the updated residual each time.

**Result:** INVALID
explainability 0.931 < 0.95
unexplained/residual 1.92 > 0.80
```

it failed On the second iteration, because the rank-1 step absorbed TX-driven components that had not been fully removed by the first TX fit.
The scorer's internal explainability check could not attribute this mixed component to either TX products or a rank-1 source, causing validity failure.



###### Attempt 2: TX → rank-1 → TX (second TX pass at the end)

I tried One rank-1 step in the middle, followed by a second TX cancellation pass to mop up any remaining IM3 leakage.

**Result:** INVALID

```
unexplained/residual 1.12 > 0.80
```

it failed because the second TX fit on the already-cleaned signal was subtracting components that did not project well onto the scorer's fixed IM3 term set. This left an unexplained residual exceeding the validity threshold.



###### Attempt 3: Rank-2 spatial cancellation

I tried to remove the top 2 eigenvectors of the spatial covariance matrix instead of just the dominant one, to capture more external interference power.

**Result:** INVALID

```
explainability 0.937 < 0.95
unexplained/residual 1.09 > 0.80
```

it failed because the scorer's validity model only accounts for a single rank-1 external component. Removing a second spatial direction produced a removed signal that could not be decomposed into TX-part + rank-1-part with sufficient accuracy (≥ 95% explainability required).

##### 

