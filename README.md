# mask-tool

### install
```
yarn add https://github.com/looxis/mask-tool

or

npm install --save https://github.com/looxis/mask-tool
```

### import component
```
import MaskTool from "mask-tool";
```
### connect component and pass image src to it
#### Props
##### src
Type: `String`<br>
Required: `true`<br>

```
<mask-tool src="..."></mask-tool>
```


### to get result image:
#### - set `ref` to MaskTool component
#### - set listener for `@export` event
#### - call: `$refs.maskTool.createExport()`
#### - resulting image passing to your `@export` event listener as Blob object

```
<mask-tool ref="maskTool" @export="handleMaskToolExport" src="..."></mask-tool>
```
