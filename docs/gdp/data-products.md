# GDP1 data products

GDP1's Slocum workflow creates files with different roles. Profile NetCDF files contain the converted observations. JSON files supply metadata for conversion, while an XML fragment can describe a dataset for ERDDAP.

| File | What it contains | What to know |
| --- | --- | --- |
| `deployment.json`, `instruments.json`, `trajectory.json` | Deployment, instrument, and variable metadata prepared from Sensor Tracker records. | These are conversion inputs, not observation files. Existing JSON can be reused on later runs. |
| ASCII files | Intermediate data converted from selected binary files or supplied as existing ASCII files. | Cleanup can remove processed files. |
| Profile NetCDF (`.nc`) | Observations written as individual profiles from eligible ASCII input. | This is the main observation output. One deployment can have many profile files. |
| ERDDAP dataset XML fragment | Dataset configuration that can be generated when the XML step is selected and a NetCDF file is available. | It is configuration, not an observation file or proof of a live dataset. |

## Reading a profile file

GDP1 restricts converted data to the deployment's recorded time window. It also leaves out data columns without definitions in the JSON metadata. The variables, units, attributes, and time coverage can therefore differ between files. Check the contents of the particular profile file before comparing it with another file or using it in an analysis.

## Quality and availability

Creating a NetCDF file does not certify its measurements or make it available through ERDDAP. GDP1's NetCDF evaluation handler is inactive, and its output metadata gives no assurance of quality control. Review the data for the intended scientific use.

When selected, separate steps can transfer the XML fragment and NetCDF files to configured destinations. Those transfers do not verify that ERDDAP is serving the dataset. Check the live service before sharing an ERDDAP link.
