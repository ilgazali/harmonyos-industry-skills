---
id: UTIL-37
title: Photo geotag map - read GPS EXIF from a picked image and drop a snapshot marker on it
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/37_image_position.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_position-0000002491645574
sample: huawei_industry_tree/15_utilities/downloads/ImagePosition.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.LocationKit", "@kit.MapKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, fileIo, fileUri, geoLocationManager, hilog, image, map, mapCommon, photoAccessHelper, site, window]
permissions: [ohos.permission.INTERNET, ohos.permission.MEDIA_LOCATION, ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0007, HW-15-0078, HW-15-0079, HW-15-0080, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you need to **put a photo on a map** - the gallery "places"
view, a travel journal that pins the shots of a trip, a field-inspection app
that proves where a picture was taken. The pattern is a chain of four
independent pieces: pick a file, read its GPS EXIF, reverse-geocode it to a
readable address, and render a marker whose icon is a live screenshot of an
ArkUI builder.

The transferable half is the last piece. `getComponentSnapshot().createFromBuilder`
turns any `@Builder` into a `PixelMap`, and `Marker.setIcon` accepts that
`PixelMap`. That means a map pin can be an arbitrary ArkUI subtree - a thumbnail
with a `Badge` count and a focus border, as here - instead of one of a handful
of static PNGs. The same trick builds custom cluster bubbles, price pins on a
property map, or avatar pins on a friend map.

**Read `HW-15-0079` before adopting the EXIF half.** The sample declares
`MEDIA_LOCATION` and never requests it, so on a real device the gallery hands
back photos whose GPS tags have been redacted and every pin lands in the same
place. The permission is the feature; without it the demo only works on files
the app wrote itself.

## Feature checklist

- On entry the app asks for the location permissions and flies the map camera
  to the user's current position at zoom 8.
- 选择图片 (select image) opens the system photo picker, limited to one image.
- The picked photo's GPS EXIF is parsed into a latitude/longitude pair.
- The coordinate is reverse-geocoded and the address is available for display.
- A marker is added at the coordinate; its icon is a rendered 64 vp thumbnail
  of the photo with a red count badge.
- Tapping a marker toggles its focus: a blue border appears and the address
  strip above the buttons shows that photo's address; tapping again clears both.
- Repeated photos from the same coordinate raise the badge count.
- 全部清除 (clear all) removes every marker and resets the maps, sets and the
  address strip. The button is disabled until at least one marker exists.

## Architecture

One `entry` module, one page and two util files. There is no model layer - the
`PicInfo` interface is the whole domain.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       full-screen window, forces COLOR_MODE_LIGHT
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                     @Entry: MapComponent, markers, picker, snapshot icons
└── utils
    ├── ImagePosition.ets               PicInfo, EXIF read, DMS -> decimal, reverseGeocode
    └── PermissionMng.ets               requestPermissionsFromUser wrapped in a Promise
```

The documented 工程目录 does **not** match the zip (`HW-15-0007`): it spells
the ability file `Entryability.ets` where the zip ships `EntryAbility.ets` - a
difference that only bites on a case-sensitive filesystem - and it omits the
`entrybackupability` directory the zip actually contains.

**The design decision worth copying** is that `getImagePosition` returns one
flat `PicInfo` carrying everything the caller needs - coordinate, source uri,
resolved address and a slot for the `Marker` handle - and the page keys three
different maps off pieces of it. `picMap` is keyed by `marker.getId()` so the
marker-click callback can recover the photo; `locMap` is keyed by the
stringified lat+lon so repeat visits to one place bump a badge instead of
stacking pins; `picSet` deduplicates by uri so re-picking the same photo does
not inflate the count. Three cheap lookups replace what would otherwise be a
linear scan over a marker array on every tap.

The trap in the same object is that `PicInfo` has **no field saying whether the
photo had a coordinate at all** (`HW-15-0080`). The no-GPS branch returns
`latitude: 0, longitude: 0` and the caller cannot tell that from a real fix in
the Gulf of Guinea, so it flies the camera to null island and drops a pin there.
Add a boolean or return `undefined`; do not encode "missing" as a valid
coordinate.

## Implementation steps

1. **Declare `INTERNET`, `MEDIA_LOCATION`, `LOCATION` and
   `APPROXIMATELY_LOCATION`** in `module.json5`, and point every `usedScene.abilities`
   entry at an ability that exists - the sample names `EntryFormAbility`, which
   this project does not have (`HW-15-0079`). Note the document's 权限说明 lists
   `GET_NETWORK_INFO` instead of `INTERNET` (`HW-15-0007`).
2. **Request all three user_grant permissions, not just the two location ones.**
   `MEDIA_LOCATION` is what un-redacts EXIF GPS on gallery photos (`HW-15-0079`).
3. **Branch on `data.authResults` before proceeding** - a refusal returns
   `authResults: [-1]` with no `err`, so the sample's `resolve(true)` treats
   "denied" as "granted" (`HW-15-0078`).
4. **Open the image by fd**: `new fileUri.FileUri(uri)` then `fs.openSync(fileObj.path, READ_ONLY)`,
   and create the `ImageSource` from `file.fd` inside a `try/finally` that closes
   the fd.
5. **Read the four GPS keys together** - `GPS_LATITUDE`, `GPS_LATITUDE_REF`,
   `GPS_LONGITUDE`, `GPS_LONGITUDE_REF` - in one `getImageProperties` call. The
   REF fields carry the hemisphere and are what turns the magnitude into a
   signed value.
6. **Convert DMS to decimal with `parseFloat`, never `parseInt`** (`HW-15-0080`),
   and surface the no-GPS case to the caller instead of returning `(0, 0)`.
7. **Reverse-geocode with `site.reverseGeocode`** using `radius: 100` and
   `language: 'zh'`, and keep the returned `addressDescription` on the `PicInfo`
   so the marker tap can display it without a second network round trip.
8. **Render the marker icon from a `@Builder`** through
   `getComponentSnapshot().createFromBuilder`, then `marker.setIcon(pixelMap)`.
   Re-run the same call with `focus: true` to repaint the border on selection.

## Verified snippets

All snippets are from `ImagePosition.zip`. Corrected forms are marked.

**The permission wrapper — `entry/src/main/ets/utils/PermissionMng.ets`**
(corrected, see `HW-15-0078`, `HW-15-0079`)

```typescript
import { abilityAccessCtrl, PermissionRequestResult, Permissions } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

export function getPermission(context: Context, permissions: Permissions[]): Promise<boolean> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  return new Promise((resolve, reject) => {
    atManager.requestPermissionsFromUser(context, permissions,
      (err: BusinessError, data: PermissionRequestResult) => {
        if (err) {
          hilog.error(0X0000, 'testTag', `requestPermissionsFromUser fail, err: ${JSON.stringify(err)}`);
          return reject(false);
        }
        // FIX: the sample resolves(true) here unconditionally
        const granted = data.authResults.every((result: number) => result === 0);
        hilog.info(0X0000, 'testTag', `requestPermissions data: ${JSON.stringify(data)}`);
        return resolve(granted);
      });
  });
}

export function getLocationPermission(context: Context): Promise<boolean> {
  // FIX: the sample requests only the two location permissions
  let requestPermission: Permissions[] = [
    'ohos.permission.APPROXIMATELY_LOCATION',
    'ohos.permission.LOCATION',
    'ohos.permission.MEDIA_LOCATION'
  ];
  return getPermission(context, requestPermission);
}
```

**`err` and refusal are different channels.** `requestPermissionsFromUser`
reports a *system* failure through `err` and the *user's answer* through
`data.authResults`, one entry per requested permission, `0` for granted and
`-1` for denied. The shipped code only looks at `err`, so the "user tapped
禁止" path resolves `true`, `moveToLocation` proceeds, and
`geoLocationManager.getCurrentLocation` rejects into a promise chain with no
`.catch` - an unhandled rejection instead of a handled state. This same defect
is filed as a systematic across three utilities samples (`HW-15-0078`).

The second correction is the whole feature. `MEDIA_LOCATION` is the permission
that lets an app read location metadata out of files it did not write; without
it `getImageProperties` returns GPS tags that the media service has stripped,
so every gallery photo looks unlocated.

**EXIF GPS to a decimal coordinate — `entry/src/main/ets/utils/ImagePosition.ets`**
(corrected, see `HW-15-0080`)

```typescript
export interface PicInfo {
  longitude: number;
  latitude: number;
  hasLocation: boolean;   // FIX: absent in the sample
  uri: string;
  address: string;
  marker: map.Marker | null;
}

export async function getImagePosition(context: Context, uri: string): Promise<PicInfo> {
  let imageSource: image.ImageSource = getImageSource(uri);
  let key = [image.PropertyKey.GPS_LONGITUDE, image.PropertyKey.GPS_LONGITUDE_REF,
    image.PropertyKey.GPS_LATITUDE, image.PropertyKey.GPS_LATITUDE_REF];
  let data = await imageSource.getImageProperties(key);
  if (!data || !data.GPSLongitude || !data.GPSLatitude) {
    return { longitude: 0, latitude: 0, hasLocation: false, uri, address: '', marker: null };
  }
  let loStr = data.GPSLongitude.split(',');
  let laStr = data.GPSLatitude.split(',');
  // FIX: the sample uses parseInt on all six components
  let longiVal = Math.round((parseFloat(loStr[0]) + parseFloat(loStr[1]) / 60 + parseFloat(loStr[2]) / 3600) * 1000) / 1000;
  let latiVal = Math.round((parseFloat(laStr[0]) + parseFloat(laStr[1]) / 60 + parseFloat(laStr[2]) / 3600) * 1000) / 1000;
  let longitude = data.GPSLongitudeRef === 'E' ? longiVal : (0 - longiVal);
  let latitude = data.GPSLatitudeRef === 'N' ? latiVal : (0 - latiVal);
  let params: site.ReverseGeocodeParams = {
    location: { latitude: latitude, longitude: longitude },
    language: 'zh',
    radius: 100
  };
  let result = await site.reverseGeocode(context, params);
  return { longitude, latitude, hasLocation: true, uri, address: result.addressDescription, marker: null };
}
```

**Two things carry this function.** The first is the REF pair: EXIF stores the
magnitude and the hemisphere separately, so `GPSLatitudeRef === 'N' ? v : -v`
is not a nicety - drop it and every photo from the southern or western
hemisphere lands mirrored across the equator or the prime meridian.

The second is the parse. The seconds component of a DMS triple is routinely
fractional (`"39,54,27.638"`), and `parseInt` silently truncates it - a third
of an arc-second, tolerable. But EXIF rationals are also legal here
(`"27638/1000"`), and `parseInt` on that yields `27638`, which is not a
truncation but garbage: it drives the whole computed latitude off by several
degrees. `parseFloat` handles the decimal form correctly; a rational form still
needs a `/` split before it is fed to either.

**The marker icon is a component screenshot — `entry/src/main/ets/pages/Index.ets`**
(as shipped)

```typescript
@Builder
MarkerIconBuilder(uri: string, cnt: number, focus: boolean = false) {
  Row() {
    Badge({
      count: cnt,
      style: { badgeColor: Color.Red },
      position: BadgePosition.RightTop,
    }) {
      Row() {
        Image(uri)
          .width(64)
          .height(64)
          .borderRadius(4);
      }
      .border({ radius: 4, width: 2, color: focus ? '#0A59F7' : Color.White });
    };
  }
  .width(80)
  .height(80);
}

setMarkerIcon(marker: map.Marker, focus: boolean = false) {
  let picInfo = this.picMap.get(marker.getId());
  if (picInfo === undefined) {
    return;
  }
  this.address = focus ? picInfo.address : '';
  let latLng = picInfo.latitude.toString() + picInfo.longitude.toString();
  let locCnt = this.locMap.get(latLng) ?? 1;
  this.getUIContext().getComponentSnapshot().createFromBuilder(() => {
    this.MarkerIconBuilder(picInfo!.uri, locCnt, focus);
  },
    async (error: Error, pixelMap: image.PixelMap) => {
      if (error) {
        hilog.error(0xFF00, 'ImagePosition', `error message is ${error.message}`);
        return;
      }
      marker.setIcon(pixelMap);
    });
}
```

**`createFromBuilder` renders a subtree that was never mounted.** The builder is
invoked off-screen, rasterised, and handed back as a `PixelMap` in the callback
- which is why the icon can contain an `Image` loaded from a photo uri and a
`Badge`, neither of which any static marker icon API would give you. The whole
selection affordance is then just a re-render: `setMarkerIcon(marker, true)`
runs the same builder with `focus: true`, which flips the border colour from
white to `#0A59F7`, and assigns the new `PixelMap` over the old one.

The `?? 1` default on `locCnt` is load-bearing in the opposite direction from
the `?? 0` in `selectImage`: on the write path an unseen coordinate starts at
zero and is stored as one, on the read path a missing entry must still render a
badge of one rather than zero.

**Picking and pinning — same file** (as shipped, with the `HW-15-0080` guard noted)

```typescript
selectImage() {
  let photoSelectOptions: photoAccessHelper.PhotoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  photoViewPicker.select(photoSelectOptions)
    .then((photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      photoSelectResult.photoUris.forEach(async (uri: string, i: number) => {
        let locInfo: PicInfo = await getImagePosition(this.getUIContext().getHostContext()!, uri);
        // FIX: `if (!locInfo.hasLocation) { return; }` belongs here - see HW-15-0080

        if (!this.picSet.has(uri)) {
          let latLng = locInfo.latitude.toString() + locInfo.longitude.toString();
          let locCnt = this.locMap.get(latLng) ?? 0;
          this.locMap.set(latLng, locCnt + 1);
          this.picSet.add(uri);
        }

        let markerOptions: mapCommon.MarkerOptions = {
          position: { latitude: locInfo.latitude, longitude: locInfo.longitude },
          icon: 'icons.png',
          clickable: true
        };
        let marker: map.Marker | undefined = await this.mapController!.addMarker(markerOptions);
        this.hasMarker = true;
        locInfo.marker = marker;
        this.picMap.set(marker.getId(), locInfo);
        this.setMarkerIcon(marker, false);

        this.moveCamera(locInfo);
      });
    }).catch((err: BusinessError) => {
    hilog.error(0xFF00, 'ImagePosition', `photoPicker failed, message is ${err.message}`);
  });
}
```

**The picker needs no permission.** `PhotoViewPicker` runs in the system photo
app's own process and hands back a uri the app is granted read access to, which
is why `READ_IMAGEVIDEO` is absent from this project entirely - and correctly
so. `MEDIA_LOCATION` is a separate axis: it governs the *metadata* inside the
file, not the right to open it.

Two rough edges to fix if you copy this: `icons.png` is a placeholder icon
overwritten a few lines later by `setMarkerIcon`, so the first frame flashes a
default pin; and `addMarker` is typed `map.Marker | undefined` but dereferenced
without a check on the very next line.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  {
    "name": "ohos.permission.MEDIA_LOCATION",
    "reason": "$string:media_reason",
    "usedScene": { "abilities": ["EntryFormAbility"], "when": "inuse" }   // no such ability
  },
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:location_reason",
    "usedScene": { "abilities": ["EntryFormAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:location_reason",
    "usedScene": { "abilities": ["EntryFormAbility"], "when": "inuse" }
  }
]
```

- All three `user_grant` entries name `EntryFormAbility` in `usedScene`; the
  module declares exactly one ability, `EntryAbility` (`HW-15-0079`).
- Only two of the three are ever passed to `requestPermissionsFromUser`.
- The document's 权限说明 lists `GET_NETWORK_INFO`, which the zip never
  declares, and omits `INTERNET`, which it does (`HW-15-0007`).
- `metadata.client_id` is present in `module.json5` - Map Kit will not render a
  tile without it, and it must be replaced with your own AGC client id along
  with a matching debug signature.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Map Kit requires the map service to be enabled in AGC and a manually
  configured signing profile; the sample will show a blank map otherwise.
- `deviceTypes` is `phone`, `tablet`, `2in1`.
- `EntryAbility.onCreate` pins the app to `COLOR_MODE_LIGHT`, so the sample
  never exercises a dark-mode map style.
- `site.reverseGeocode` is a network call per photo with no cache and no
  batching; picking many photos issues one request each.
- The picker is capped at `maxSelectNumber = 1`, but `selectImage` already
  iterates `photoUris`, so raising the cap needs no other change - except that
  the per-photo `moveCamera` would then fight itself.

## Pitfalls

- **`HW-15-0078`** (D/high, confirmed): `getPermission` resolves `true` whenever
  `err` is absent, so a refusal - which returns `authResults: [-1]` and no
  error - is treated as a grant; the location call that follows then rejects
  into a chain with no `.catch`. Filed as a systematic: the same shape appears
  in `LocationData` and `ImageDecoder`. Fix: branch on `data.authResults`.
- **`HW-15-0079`** (D/medium, confirmed): `MEDIA_LOCATION` is declared but never
  requested, so gallery photos come back with their GPS EXIF redacted and the
  core feature silently fails on every real photo; `usedScene.abilities` also
  names a nonexistent `EntryFormAbility`. Fix: add it to the requested array and
  point `usedScene` at `EntryAbility`.
- **`HW-15-0080`** (B/medium, confirmed): DMS components are parsed with
  `parseInt`, truncating fractional seconds and mangling rational EXIF values
  outright; separately, photos with no GPS return `(0, 0)` with no flag, so the
  camera flies to null island and drops a marker there. Fix: `parseFloat` plus
  an explicit `hasLocation`.
- **`HW-15-0007`** (E/low, confirmed): the document's 权限说明 lists
  `GET_NETWORK_INFO`, which the sample never declares, while the sample declares
  `INTERNET`, which the document never mentions; the 工程目录 spells the ability
  file `Entryability.ets`. The referenced 图片获取与保存实践 page is also absent
  from the site navigation. Fix: sync both lists and correct the filename.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `getImageProperties`, `PropertyKey`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-guides/05_media/component-guidelines-photoviewpicker.md` - `PhotoViewPicker.select` and `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/component-guidelines-photoviewpicker
- `documentation/harmonyos-guides/03_application-framework/arkts-uicontext-component-snapshot.md` - `createFromBuilder` and its callback contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-uicontext-component-snapshot
- `documentation/harmonyos-references/03_application-services/map-site.md` - `site.reverseGeocode` and `ReverseGeocodeParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-site
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - `MarkerOptions`, `setIcon`, `markerClick`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - `authResults` and the refusal path
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `MEDIA_LOCATION`, `LOCATION`, `APPROXIMATELY_LOCATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene` rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Kit and the client id
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `UTIL-38` - the other half of the story: writing the geotag at capture time
