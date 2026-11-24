# GDP2 data products

GDP2 creates several kinds of files for Slocum deployments. Each has a different purpose. Creating a NetCDF file does not, by itself, make it available through ERDDAP or show that its measurements suit a scientific analysis.

| Product | What it contains | Use it for |
| --- | --- | --- |
| `deployment.yml` | Deployment metadata, glider device details, NetCDF variable mappings, and profile variable definitions. | Check the information supplied to `pyglider` for conversion. It is not an observation file. |
| Time-series NetCDF (`.nc`) | Slocum observations converted from selected raw files and restricted to the deployment time window. | Work with the deployment as a time series. |
| Profile NetCDF files (`.nc`) | Individual profiles extracted from the time-series NetCDF under the profile layout. | Work with profile-based observations. |
| `report.json` | Metadata and data-processing errors GDP2 caught while handling selected deployments, grouped by issue and deployment. | Investigate processing problems. It is not a quality-control report. |

## Choosing a NetCDF file

A full-conversion time-series file represents the selected deployment input on a time axis. A realtime batch file is generated from new pairs, with the latest previous pair included as conversion context when available; its extracted profiles are merged into the existing collection. GDP2 prepares profiles under its default layout during processing, even if ERDDAP publication is not requested. A configured delayed-deployment layout can instead use the time-series file for publication.

The variables in a NetCDF file depend on the deployment's metadata mappings. Check the variables, units, attributes, and time coverage in the particular file before comparing it with another deployment. Do not assume a fixed variable list or identical coverage across deployments.

## Quality and availability

When report-mode checks are enabled, GDP2 can log depth comparisons and position summaries. These are diagnostics for review, not a pass/fail judgement or an assessment of whether the data suit a particular analysis. Their findings are not recorded in `report.json`, which lists caught metadata and data-processing errors. Review the file and relevant diagnostics before using its measurements.

ERDDAP publication requires an explicit request. A profile file, for example, can exist in GDP2's output without appearing on ERDDAP. When publication is requested and configured, GDP2 copies eligible files, updates the dataset configuration, and requests a reload when needed. The reload request does not confirm that the live service is serving the dataset; check ERDDAP before sharing its link.
