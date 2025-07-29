# SPLUNK-Extract-Employee-Details
This Repository contains the code to extract the overall scanning done by the HUB in order to extrapolate the information to do the individual KPI

/*
# 📦 UDS Barcode Scan Analysis - ZLXH Site
This Splunk query retrieves and correlates barcode scan data from UDS applications and equipment logs at site ZLXH.
It extracts and matches LoadingUnit and HandlingUnit barcodes with additional trailer metadata such as asset ID,
trailer type, tracking number, and scan details. 

Author: Abdur Rafeh Kayani
Last Updated: 29/07/2025

Key Objectives:
- Extract barcodes (LU/HU) and scan metadata from JSON message logs.
- Join against validation event logs to enrich with trailer asset ID (LuAssetId).
- Infer trailer type and extract tracking numbers using heuristic rules.
*/

index=uds_application OR index=uds_equipment sourcetype=* site=ZLXH host=GBZLXA-FLC

// Extract fields from the main JSON log structure
| eval loading_unit = coalesce(
      json_extract(Message,"LoadingUnit.Barcode"),
      json_extract(Message,"Barcode")                     // Fallback if LU Barcode is not under LoadingUnit
  )
| eval handling_unit     = json_extract(Message,"HandlingUnit.Barcode")
| eval init_ScanDateTime = json_extract(Message,"ScanDateTime")
| eval init_ScanUserID   = json_extract(Message,"ScanUserId")
| eval init_DeviceID     = json_extract(Message,"DeviceId")
| eval init_Destination  = json_extract(Message,"LoadingUnit.Destination")
| rename source as init_source

// Only consider logs that have ScanUserId (i.e., valid scans)
| where isnotnull(init_ScanUserID)

//////////////////////////////////////////////////////////////////////////////
// 🔁 Join with validation logs to retrieve LuAssetId from trailer scan events
//////////////////////////////////////////////////////////////////////////////
| join type=left loading_unit handling_unit [
    search index=uds_application OR index=uds_equipment sourcetype=uds.core.log site=ZLXH host=GBZLXA-FLC
    Event IN ("RfLoadingValidationWithData", "RfLoadingValidationRequestFor
