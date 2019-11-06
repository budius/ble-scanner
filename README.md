# A Sane Android BLE scanner

Builds on top of the excellent NordicScannerCompat to create a sane to use ble-scanner.
Internally counts start/stop to avoid Nougat ble-scan limit.
Provides separation between scanning and listening for scans.
Properly checks runtime permissions and bluetooth status, and start scanning after it's all OK.

Example:
- bluetooth is off
- app calls `bleScanner.start()`, it doesn't start yet,
- user then turn on bluetooth, then it starts scanning.

[ ![Download](https://api.bintray.com/packages/sensorberg/maven/ble-scanner-core/images/download.svg) ](https://bintray.com/sensorberg/maven/ble-scanner-core/_latestVersion)

### Adding to your project

```
// the base scanner
implementation 'com.sensorberg.libs:ble-scanner-core:<latest>'

// start/stop scan based on a lifecycle or the process lifecycle
implementation 'com.sensorberg.libs:ble-scanner-lifecycle:<latest>'

// Extra helpers, such as list of "nearby" scans and scanFinder
implementation 'com.sensorberg.libs:ble-scanner-extensions:<latest>'
```

## How to use

### Initialise the scanner
```
fun initScanner(context: Context, backgroundHandler: Handler): BleScanner {
	return BleScanner
			.builder(context)
			.handler(backgroundHandler) // used for timing operations
			.filters(listOf(MY_DEVICE_FILTER))
			.build()
}
```

### Start/Stop the scanner
```
bleScanner.start()
bleScanner.stopDelayed() // use this to mitigate Nougat ble-scan limits
bleScanner.stopNow() // if really need everything to stop now (example: to execute Gatt operations)

// or even better
bleScanner.scanWithProcessLifecycle() // scans while the app is in foreground
```
Q: Why this is better than Android framework?

A: Duplicate calls to start (or to stop) or start while bluetooth off or without permission, none of it is a problem.
   Application can call any of it at any moment, the scanner simply will use the last request.


### Get the status of the scanner.
```
val scannerState: LiveData<BleScanner.State> = bleScanner.getState().toLiveData()
scannerState.observe(this, Observer {
	when (it) {
		is BleScanner.State.IDLE -> {
			when (it.why) {
				NO_SCAN_REQUEST -> // application didn't call start
				BLUETOOTH_DISABLED -> // bluetooth is off
				LOCATION_DISABLED -> // location is off
				NO_LOCATION_PERMISSION -> // app didn't get location permission
			}
		}
		BleScanner.State.SCANNING -> Timber.d("Scanning")
		is BleScanner.State.ERROR -> Timber.d("Scan error ${it.errorCode}")
	}
})
```

Q: Why this is better than Android framework?

A: One central location to get information on what's going on.
   No need to fuzz with LocationManager or the runtime permissions

### Listen for scans
```
val callback = object : ScanResultCallback {
    override fun onScanResult(scanResult: ScanResult) {

    }
}
bleScanner.addCallback(callback)
bleScanner.removeCallback(callback)
```

Q: Why this is better than Android framework?

A: Simpler callback, Better separation of concerns as the callbacks can be added or remove at anytime and independent from start/stop

### Listen for list of nearby scans

Use `ble-scanner-extensions`

```
val cancellation = Cancellation()

val bleScanList: LiveData<List<BleScanResult>> = ObservableBleScanResult
        .builder(bleScanner)
        .handler(backgroundHandler) // re-use the handler from the scanner
        .cancellation(cancellation) // stops observing the bleScanner when `cancel` is called
        .timeoutInMs(7_500L) // when devices are not "nearby" anymore
        .debounceInMs(1000) // avoid overloading the UI thread with updates
        .averagerFactory { scanResult ->
            if (isDeviceX(scanResult)) {  // averager from https://github.com/sensorberg-dev/motionless-average
                createAveragerForDeviceX()
            } else {
                createOtherAverager()
            }
        }
        .build()
        .toLiveData()
```

Q: Why this is better than Android framework?

A: The framework doesn't have anything as sophisticated as that.
   - keep nearby devices using customizable timeout
   - average RSSI built-in
   - debouce for UI thread update

### Looking for specific device/scan

Use `ble-scanner-extensions`

```
// map from ScanResult to any generic T you might need, String is just an example
val scanMapper: ((ScanResult) -> String?) = { scanResult ->
    if (isTheDeviceIamLookingFor(scanResult)) {
        scanResult.device.address
    } else {
        null
    }
}

// find the device
scanResultFinder(
        bleScanner,
        cancellation, // to stop the process when needed
        bleScanList.value?.map { it.scanResult }, // if you have any values in a "nearby" list
        scanMapper) {
    Timber.d("Found the device $it")
}
```