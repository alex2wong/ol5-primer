# using Openlayers in Vue.js app

在`vue.js`中使用`Openlayers`库是非常方便的，因为最新的Openlayers5.x 已经采用ES6开发，可以顺畅地引入ES6或者Typescript开发的现代前端项目中。本章节主要讲解在vue.js app中封装使用Openlayers库涉及到的一些关键问题，分为以下几块内容：

- 在vue.js中使用Openlayers
- 数据驱动地图视图
- 利用插槽实现组件嵌套

## Vue.js中使用Openlayers

参考ol官网示例和vue.js官网教程，我们可以快速地在Vue.js项目中引入ol，并且构建一个vue组件

```html

<!-- 在 map-component.vue 文件中定义组件 -->
<template>
    <div ref="mapCont" class="v-map__cont"></div>
</template>
<script>
import Vue from 'vue'
import 'ol/ol.css';
import Map from 'ol/Map';
import OSM from 'ol/source/OSM';
import TileLayer from 'ol/layer/Tile';

export default Vue.extend({
    props: {
        center: {
            default() { return [0, 0]; }
        },
    },
    mounted() {
        const map = new Map({
            center: this.center,
            zoom: 1,
            target: this.$refs.mapCont,
            layers: [
                new TileLayer({
                    source: new OSM()
                })
            ],
        });
    }
})
</script>

<!-- 使用组件 -->
<template>
    <v-map :center="center"></v-map>
</template>
<script>
import Vmap from './map-component.vue';
export default Vue.extend({})
</script>

```
完成第一段代码所示的ol组件后，我们可以在项目中引入这个组件并用声明式的语法(`<v-map/>`)去使用，来完成地图的实例化和参数配置。这使得封装完 map 组件后，在项目中可以比较容易地复用，不再需要繁琐的 `new Map({...})`。这还不是最关键的，借助 vue.js 框架的响应式特征，我们可以用数据驱动地图视图的重新渲染。

## 数据驱动地图视图

在现代前端项目的开发中，我们需要尽量做到数据变动自动驱动视图的变化，例如地图中心变动了，要素点增加或者坐标变动了，只要事先绑定好数据和地图视图的关系，就可以做到自动重新渲染。相反的，我们拖动地图，选中要素，很可能需要相应触发数据的更新，进而触发其他表格、列表等视图的变化。

那么在Vue.js中，可以利用`watch`或计算表达式和`event`来完成双向数据绑定。

```javascript
// v-marker.vue
export default Vue.extend({
    props: {
        lnglat: {
            default() { return [0, 0] }
        },
    },
    watch: {
        lnglat: function(newlnglat) {
            if (this.findParentLayer !== undefined) {
                const lnglat2proj = transform(newlnglat, 'EPSG:4326', 'EPSG:3857');
                this.marker.getGeometry().setCoordinates(lnglat2proj);
            }
        },
    }
}
// emit feature dragEvent, just for demo.
function dragFeature(evt) {
    this.$emit('update:lnglat', evt.feature.getGeometry().getCoordinates());
}

// 在使用v-marker的时候，就实现了双向绑定
<v-marker :lnglat.sync="coord" :name="'marker1'" />
```
以上代码演示的是利用`watch`函数去监听传入的marker坐标变化，继而更新图层中geometry的坐标。相反的，在地图视图中拖拽这个feature，我们可以把对lnglat的更新emit出去，vue.js 默认接受`update:[propName]`作为双向绑定的语法，只要在使用时用[`.sync`修饰符](https://cn.vuejs.org/v2/guide/components-custom-events.html#sync-%E4%BF%AE%E9%A5%B0%E7%AC%A6)就可以启用。如此外层的coord变量就会随着要素拖动而发生变化。 .sync修饰符的使用等同于以下语法：
```html
<v-marker :lnglat="coord" @update:lnglat="coord=$event">
```

## 利用插槽实现组件嵌套

完成了地图组件的简单封装和数据驱动视图，还有一个问题没有解决：在原生的前端WebGIS项目中我们有各种办法去解决`map实例、layers以及markers等的从属关系`，也就是用合适的办法把点线数据加到对应的图层实例中。在一个图层的情况下还好办，假如我们有几个地图实例，并且他们都有不止一个图层呢？Vue.js项目中使用组件的声明式语法如何应对呢？

```html
<v-map mapId="map1" :center="center" :zoom="2">
    <tile-layer></tile-layer>
    <vector-layer url="../static/places_simple.json">
        <v-marker></v-marker>
    </vector-layer>
</v-map>
```
在`VectorLayer`组件的实现中，我们需要在矢量图层实例化后拿到所对应的map实例，调用`map.addLayer`方法。如果你了解Vue.js，应该明白这首先需要VMap接受插槽来获得子组件分发的内容。其次，`VectorLayer`组件中需要获得父组件中map实例。

```Javascript
// VectorLayer组件的实现
mounted() {
    const vectorlayer = new VectorLayer(
        Object.assign({}, { source, style }, this.$props.options)
    )
    Store.setLayer(this.layerId, vectorlayer);
    this.layer = vectorlayer;
    // 获取Vectorlayer的父组件map实例
    if (vectorlayer && this.findParentMap !== undefined) {
        this.$nextTick(() => {
            Store.addLayer(this.findParentMap().$props.mapId, vectorlayer);
            this.$emit('vlayer-added', this.layerId);
        });
    }
},
methods: {
    findParentMap() {
        if (this.$parent instanceof VMap) {
            return this.$parent;
        } return undefined;
    }
},
```

通过上面的方式，就可以实现每个`VectorLayer`都能找到自己对应的Map实例。

**注意**
vue.js 官网文档中不建议过多采用`this.$parent`，而应该尽量采用`prop`和`event`来传递消息。

### 总结
至此，我们已经简单实现了vue.js中对Openlayers组件的封装，这使得我们可以复用封装好的各类组件，并且通过在template中的声明式语法来嵌套使用`map、layer和marker`等要素，传入组件的数据可以响应式的驱动地图视图的渲染，与地图要素的交互也可以反馈给组件的数据，实现联动。各位读者可以在项目中根据需求自行封装Openlayers的部分组件，也可以采用功能相对全面的vue wrapper，例如 https://github.com/ghettovoice/vuelayers/

各位读者有问题可以与本章作者讨论 [alex2wong](https://github.com/alex2wong)，[邮件地址](huangyixiu188@hotmail.com)，任何遗漏，不准确之处请与章节作者联系，谢谢。