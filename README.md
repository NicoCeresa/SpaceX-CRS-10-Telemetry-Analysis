# SpaceX-CRS-10-Telemetry-Analysis

Analysis of SpaceX CRS-10 (Commercial Resupply Service) telemetry data to understand rocket flight dynamics and identify significant events.

## Notebook Contents: `SpaceX_CRS_10_Telemetry.ipynb`

### Anomaly Detection
- **Z-score Method**: Identifies acceleration spikes exceeding 2.5 standard deviations from mean
  - Highlights unusual acceleration events on both acceleration and velocity plots
- **IQR Method**: Uses interquartile range to detect outliers beyond 1.5×IQR bounds
  - Complements Z-score detection with different statistical perspective

### Data Source
- Data from: https://github.com/shahar603/Telemetry-Data/tree/master/SpaceX%20CRS-10
