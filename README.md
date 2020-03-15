# React Native Async Load Bundle

This is an example project to build the common bundle file and the differential bundle file using metro, and load the differential bundle asynchronously in app. Compare with loading the official bundle file synchronously, there was 30% ~ 50% decrease in the load time of react view by using loading the differential bundle asynchronously.

## 📋 Contents

- [Background](#-background)
- [Usage](#-usage)
- [Experimental Data](#-experimental-data)
- [How dotes it work](#-how-dotes-it-work)
- [Contributing](#-how-to-contribute)
- [License](#-license)

## 📋 Background

As we all known that there are three parts in a offical bundle file: **Ployfill**、 **Modules**、 **Require**. If you build two different offical bundle files, you will find that there are many repeated content, which is close to 500K. In order to minimize the bundle file, we define a **common bundle file**, which only includes some basic modules(such as `react` and `react-native`). And we define a **differential bundle file**, which only includes your custom code.

Before React Native 0.55, we generally use `google-diff-match-tool` or `BSDiff` to build the differential bundle file, which needs the process of merging before your app loading the differential bundle file.

However, there is a new way to build the differential bundle file by using metro.

## 📋 Usage

1. Modify the `common.js`

```javascript
/**
 * Example of common.js
 */
require("react-native");
require("react");
// Add other libs you want to add to the common bundle like this:
// require('other');
```

2. Build the common bundle file with `--config metro.config.common.js` or use the command blew:

```shell
# for android:
npm run build_android_common_bundle
# for ios:
npm run build_ios_common_bundle
```

3. Build the differential bundle file with `--config metro.config.diff.js` or use the command blew:

```shell
# for android:
npm run build_android_index_diff_bundle
# for ios:
npm run build_ios_index_diff_bundle
```

4. Copy all output files to the dir of app project or use the command blew:

```shell
npm run copy_files_to_projects
```

5. Run the app project by Android Studio or XCode.

> There are two ways to start an activity with react naitve in android app: one as sync, the other as async. It is same with the offical reference implementation when using sync. As for async, it will start a general activity, which will load a common bundle file, after that it will start a custom activity using react native, which will only load the differential bundle file. The load time of react view will display by log and toast. If you want to get the load time accurately, you should restart the app before clicking one of the bottom two buttons.

Note: The `license` badge image link at the top of this file should be updated with the correct `:user` and `:repo`.

## 📋 Experimental Data

### 1. Compare the size of output file

### 2. Compare the load time of react view.

## 📋 How dotes it work

### 1. Build a differential bundle file using metro.

The key to build a differential bundle file is making the id of input module invariant during the process of bundling. It is noteworthy that the metro provides two configuration items in `metro.config.js` file: `createModuleIdFactory(path)` and `processModuleFilter(module)`.

By customizing `createModuleIdFactory(path)`, we used the **hashcode** of the file as the key to allocate module id.

```javascript
// See more code int metro.config.base.js
// ... other code ...
function getFindKey(path) {
  let md5 = crypto.createHash("md5");
  md5.update(path);
  let findKey = md5.digest("hex");
  return findKey;
}
// ... other code ...
```

In order to avoid duplication of allocation of module id, we use a local file (`repo_for_module_id.json`) to store the result of allocation during the process of buliding.

```json
// ... other data ...
"8b055b854fd2345d343b6618c9b71f7e": {
    "id": 5,
    "type": "common"    // it means the module is included in common bundle file
    },
"51a407ce4b86f3682b6d252065073d57": {
    "id": 6,
    "type": "common"
    },
// ... other data ...
```

By customizing `processModuleFilter(module)`, we compare the hashcode of input `module` with localstorage. If input `module` is included by common bundle file, it will be filtered and will not be writeen to the output bundle file.

```javascript
// See more code int metro.config.base.js
// ... other code ...
buildProcessModuleFilter = function(buildConfig) {
  return moduleObj => {
    if (buildConfig.type == BUILD_TYPE_DIFF) {
      let findKey = getFindKey(moduleObj.path);
      let storeObj = moduleIdsJsonObj[findKey];
      if (storeObj != null && storeObj.type == BUILD_TYPE_COMMOM) {
        return false;
      }
      return true;
    }
    return true;
  };
};
// ... other code ...
```

However, the pollyfills is also writeen in the output bundle file after running metro. We should remove those code by ourselives. For example, we made a script file call `removePollyfill.js` in the dir `__asyc_load_shell__`, you can use it by run:

```javascript
node ./__async_load_shell__/removePolyfill.js  {your_different_bundle_file_path}
```

### 2. Load the differential bundle file asynchronously in android.

Beacause the common bundle file includes all basic codes, we should make sure a good timing to load the common bundle file before loading the differential bundle file. In the demo app, we build a guide activity to load the common bundle file, this activity is also used to simulate a PARENT activity of the activity using react native, this activity can usually
be dislayed the entrance of your business which was builded by react native in your official app.

All related code was organized in package `com.marcus.rn.async`. There are some key points about implementation:

1. We use the `ReactNativeHost` to point the path of common bundle file, and call `getReactNativeHost().getReactInstanceManager().createReactContextInBackground();` to initialize the context of React Native and load the common bundle file.
2. In order to get approximate finish time of loading common bundle file, we use `ReactMarker.addListener()` to add custom listener, and monitor the event called `ReactMarkerConstants.NATIVE_MODULE_INITIALIZE_END` to indicate the finish of common bundle file.
3. We redefine `ReactActivityDelegate` class to suit the secen of loading asynchronously. which can be found by name with `AsyncLoadActivityDelegate.java`.
4. Because the guide activty and the container activity of react native are shared the same `AsyncLoadActivityDelegate` object, we build a **singleton** class called `AsyncLoadActivityDelegateProvider` to provider the object.
5. The **load time** of react view will be displayed by log and toast. it records time period from clicking entry button to monitoring the event called `ReactMarkerConstants.CONTENT_APPEARED`.

### 3. Load the differential bundle file asynchronously in iOS.

## 📋 Contributing

See [the contributing file](CONTRIBUTING.md)!

PRs accepted.

Small note: If editing the Readme, please conform to the [standard-readme](https://github.com/RichardLitt/standard-readme) specification.

## 📋 License

[MIT © Richard McRichface.](../LICENSE)
