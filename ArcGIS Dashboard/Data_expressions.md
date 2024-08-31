# Percentage calculation for a serial chart using data expression on two different hosted feature layers.



```js
// Establish a connection to the ArcGIS Online portal
var portal = Portal('https://cdmsmith.maps.arcgis.com/');

// Load the feature set for points using a specific item ID, and select 'PWSID' and 'Tier' fields
var pointfs = FeatureSetByPortalItem(portal, '53c93c01869047348491bb6d65c96edb', 0, ['PWSID', 'Tier'], false);

// Load the feature set for a table using another specific item ID, and select 'PWSIDConcat' and 'No__Samples_per_Compliance_Peri' fields
var tablefs = FeatureSetByPortalItem(portal, 'ee4fed8f87394af8b7b1356ba0a2e832', 0, ['PWSIDConcat', 'No__Samples_per_Compliance_Peri'], false);

var features = [];

// Group the point features by 'PWSID' and count the occurrences of 'Tier' and 'PWSID'
var groupedPointfs = GroupBy(pointfs, ['PWSID'], [
    { name: 'TierCount', expression: 'Tier', statistic: 'COUNT' },
    { name: 'PWSIDCount', expression: 'PWSID', statistic: 'COUNT' }
]);

// Loop through each record in the table feature set
for (var t in tablefs) {
    var tableID = t["PWSIDConcat"];
    
    // Filter the grouped point features to match the current 'PWSIDConcat' from the table
    var pointfsFiltered = Filter(groupedPointfs, "PWSID = @tableID");
    
    var tierCount = 0;
    var pwsidCount = 0;
    
    // If there are matching point features, get the first one (since we grouped, there is only one per 'PWSID')
    if (Count(pointfsFiltered) > 0) {
        var p = First(pointfsFiltered);
        tierCount = p["TierCount"];
        pwsidCount = p["PWSIDCount"];
    }
    
    // Get the number of samples per compliance period from the table
    var daystotal = t["No__Samples_per_Compliance_Peri"];
    
    // Calculate the completion percentage
    var completionPercentage = 0;
    if (daystotal > 0) {
        completionPercentage = (pwsidCount / daystotal) * 100;
    }
    
    // Create a new feature with the calculated attributes
    var feat = {
        attributes: {
            FeatureID: tableID,
            tiercount: tierCount,
            TotalSamples: daystotal,
            PWSIDCount: pwsidCount,
            CompletionPercentage: completionPercentage
        }
    };
    
    // Add the new feature to the features array
    Push(features, feat);
}

// Define the schema and structure for the output feature set
var joinedDict = {
    fields: [
        { name: "FeatureID", type: "esriFieldTypeString" },
        { name: "tiercount", type: "esriFieldTypeInteger" },
        { name: "TotalSamples", type: "esriFieldTypeInteger" },
        { name: "PWSIDCount", type: "esriFieldTypeInteger" },
        { name: "CompletionPercentage", type: "esriFieldTypeDouble" }
    ],
    geometryType: '',
    features: features
};

// Return the resulting feature set
return FeatureSet(joinedDict);
```