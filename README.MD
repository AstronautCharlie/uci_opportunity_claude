Data preparation steps:
1. Converted .dat files to .parquet files (`convert_to_parquet.py`)
2. Renamed duplicate `accX` columns ending with `accX_2` to `accY` and `accX_3` to `accZ` (`rename_acc_cols.py`)

Experimental setup: 
* Claude Code version 2.1.59
* Sonnet 4.6
* Always cleared context between runs to avoid contamination

Test prompts and results
* Prompt 1: "Read through all of the ADL.parquet files and find relationships between the low-level sensor data and the activity columns ("Locomotion", "HL_Activity", "LL_Left_Arm", "LL_Left_Arm_Object", "LL_Right_Arm", "LL_Right_Arm_Object", "ML_Both_Arms")"
    * Prompt is intended to create a floor of performance - no helpful info about what kinds of relationships are to be looked for, or what the sensor data tracks.
    * Resulting file is `analyze_adl.py` which does the following:
        * performs an erroneous one-way F-test to identify the sensors that are most discriminative per label (top 10), but doesn't indicate relationships to specific classes
        * calculates the mean of the top discriminative sensors for each activity class
        * listed the overlap in most "informative" sensors as dictacted by F-test across all high-level activities
        * counted the number of sensors of each type (e.g. accelerometer) among the top most informative per high-level activity
    * Summarized results, giving me a table of the F-statistics of top sensors for each high level metric (but not per-activity level)
    * Notes:
        * F-test is erroneous because it doesn't correctly compute degrees of freedom and because it doesn't account for the fact that the groups are of different sizes (different numbers of incidences tagged with each activity class). Claude suggested that this is a bad approach because "the data is in time series and the classes are not independent" - need to follow up on those 
        * Claude dropped data where rows are null - in practice we want to account that data being missing may be a signal 
        * Claude understood that the MILLISEC column isn't a sensor column 

* Prompt 2: "Read through all of the ADL.parquet files and identify which values in the sensor readings are most predictive of each class in the activity columns ("Locomotion", "HL_Activity", "LL_Left_Arm", "LL_Left_Arm_Object", "LL_Right_Arm", "LL_Right_Arm_Object", "ML_Both_Arms"). Any column that is not an activity column is a sensor reading."
    * Prompt is intended to find which sensor readings are most predictive of each activity class (i.e., when to predict a given class based on the values of sensor readings), without providing any additional information about the nature of the sensor data. 
    * Resulting file is `analyze_predictive_sensors.py` which does the following: 
        * trained random forest classifiers for multi-class prediction across all classes for each activity column
        * trained random forest classifiers for binary prediction (1 vs. rest) across all classes for each activity column 
    * Summarized top sensors for each activity class within each high-level activity, telling me the median imputation assumption and that it had created the multiclass and one-vs-rest RF classifiers
    * Notes
        * No information on how the directionality of feature values affects the prediction
        * Imputed missing values with medians for each column - we want to treat missing values as a possible signal
        * Skipped trying to classify high level actions with fewer than 100 samples
        * Chose to balance class weights to account for imbalance - we may not want this as it could obscure Bayesian priors about likelihood of classes
        * Number of top sensors returned varied without apparent logic or importance cutoff
        * Sometimes aggregated sensors across axes (e.g. KNIFE2_accX, KNIFE2_accX_2, KNIFE2_accX_3 all became KNIFE2), even when not all X/Y/Z axes were present
        * Sometimes failed to incldue top sensor in printout (e.g. LOCATION_TAG4_X for Early Morning) or renamed sensors (e.g. SPOON_accX_3 became SPOON_accX for Early Morning when those are different (X_3 is actually the Z axis) - we need to make sure that column names are descriptive enough to not confuse the agent)
        * Sometimes abbreviated sensor names
        * Ordering of Top Sensors in the summary sometimes didn't match the importance scores
        * Sometimes skipped including a sensor with a higher importance in the 'Top Sensors' summary (e.g. Accelerometer_HIP_accY in Relaxing)
        * In the case of LL_Left_Arm, it created an explanation for why the top sensors were relevant ('arm raising to mouth')
        * Skipped some classes in LL_Left_Arm altogether (cut, clean), possibly because these had fewer samples, but enough to make predictions
        * Aggregated together the open/close actions of the LL_Left_Arm activity 
        * Didn't include LL_Right_Arm in its summary (LL_Left_Arm only)
        * Gave me a summary of insights ("Cross-cutting patterns")
            * Correctly identified that in object cases, object accelerometers give strong signals, but told me that this included fridge and dishwasher, which for right arm had intertial measurement units as top sensors
            * What to check on Insight summary: 
                * "Object accelerometers are the strongest signal when objects carry sensors" - What proportion of top 3 sensors are acelerometers for objects? 
                * "Location tags dominate spatial activities" - how many activities is this true for? 
                * "IMU magnetometers (not accelerometers) matter most for arm activities" - look at ML_Both_Arms, LL_Right_Arm(_Object), LL_Left_Arm(_Object) and see how often this is true
                * "Shoe IMU angular velocity is unique to Walk" - do any other classes use Shoe IMU in their top 10?
                    * This isn't true - angular velocity for shoe shows up in top 10 for stand as well, and even then other shoe sensor readings show up for other classes, so this isn't a useful insight 
                * "Back/hip accelerometers identify posture" - What does this include? lie, sit, stand, open/close drawers, open/close dishwasher
                * "Gyroscopes on upper/lower arm identify discrete gestures (reach, release, stir)while accelerometers identify gross posture" - what counts as gross posture? 
* Prompt 3: (See `prompt_3/prompt_3_output.txt` for text of the prompt because it's long)
    * Prompt gives a lot of information about what the agent is receiving, and asks for an interpretable, physical understanding of how to track locomotion from the sensor data. 
    * Ran a lot of code but didn't write it to scripts anywhere (will need to instruct it to do that in order to reproduce work)
    * Came up with a simple decision logic to categorize locomotion, doing repeated one-vs-rest binary classification
    * Provided robust physical explanation of why it used the sensors it did
    * Correctly intuited how the accelerometer data worked - need to verify how the IMUs worked
    * To check: are its calculations of metrics correct? How good is the logic it gives? 
