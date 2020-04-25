<template>
  <div class="hello">
    <div>Homies: {{ coins }}</div>
    <div id="map"></div>
  </div>
</template>

<script lang="ts">
import axios from 'axios';

import Map from 'ol/Map.js';
import View from 'ol/View.js';
import { Draw, Modify, Snap, DragBox, defaults } from 'ol/interaction';
import { Tile as TileLayer, Vector as VectorLayer } from 'ol/layer';
import { XYZ, OSM, Vector as VectorSource } from 'ol/source';
import { Circle as CircleStyle, Fill, Stroke, Style, Text } from 'ol/style';
import GeometryType from 'ol/geom/GeometryType';
import { fromLonLat, transform } from 'ol/proj';
import Polygon from 'ol/geom/Polygon';
import Feature from 'ol/Feature';
import {
  getTopLeft,
  getTopRight,
  getBottomRight,
  getBottomLeft,
} from 'ol/extent';
import { shiftKeyOnly } from 'ol/events/condition';

import { Component, Prop, Vue } from 'vue-property-decorator';

import concave from '@turf/concave';
import convex from '@turf/convex';
import bezierSpline from '@turf/bezier-spline';
import centroid from '@turf/centroid';
import bbox from '@turf/bbox';
import transformScale from '@turf/transform-scale';
import { randomPoint } from '@turf/random';
import { point, featureCollection, lineString, polygon } from '@turf/helpers';
import Point from 'ol/geom/Point';
import Circle from 'ol/geom/Circle';
import lineChunk from '@turf/line-chunk';

@Component
export default class MapWorld extends Vue {
  private map?: Map;
  private source?: VectorSource;
  private area?: VectorSource;
  private snap?: Snap;
  private areaFeature: any;
  private units: any;
  private selectedUnits: any[] = [];

  public coins = 0;
  public properties: any[] = [];
  public propertiesCoordinates: any[] = [];

  public globalCircle: any;

  data() {
    return {};
  }

  mounted() {
    this.getGeolocation();

    setInterval(() => {
      let coins = 0;
      this.properties.forEach(olId => {
        const feature = this.source.getFeatureByUid(olId);
        if (feature) {
          let count = feature.get('count');
          const max = feature.get('max');

          if (count === max) {
            coins += count;
            return;
          }

          const polygonCoordinates = (feature.getGeometry() as any).getCoordinates()[0];
          const pointCoordinates = randomPoint(1, {
            bbox: bbox(polygon([polygonCoordinates])),
          }).features[0].geometry.coordinates;
          const circleFeature = new Feature({
            geometry: new Circle(pointCoordinates, 2),
            name: olId,
          });
          this.units.addFeature(circleFeature);

          count += 1;
          feature.set('count', count);
          coins += count;
        }
      });
      this.coins = coins;
    }, 1000);
  }

  private getGeolocation(): void {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(this.onSuccess, this.onError);
    }
  }

  private onSuccess(position: any): void {
    const latitude = position.coords.latitude;
    const longitude = position.coords.longitude;
    this.initMap(longitude, latitude);
  }

  private onError(): void {
    console.log('error');
  }

  private initCentralMarker(lon: number, lat: number) {
    const circle = new Style({
      fill: new Fill({
        color: 'rgba(219,192,144, 0.5)',
      }),
      stroke: new Stroke({
        color: 'rgb(219,192,144)',
        width: 2,
      }),
    });

    const feature = new Feature({
      geometry: new Circle(fromLonLat([lon, lat]), 100),
      name: 'circle',
    });

    feature.setStyle(circle);
    this.globalCircle = feature;
    this.source.addFeature(this.globalCircle);
  }

  private initMap(lon: number, lat: number): void {
    const raster = new TileLayer({
      source: new OSM({
        cacheSize: 4000,
        wrapX: true,
      }),
    });

    this.source = new VectorSource();
    const vector = new VectorLayer({
      source: this.source,
      style: new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)',
        }),
        stroke: new Stroke({
          color: '#ffcc33',
          width: 2,
        }),
        text: new Text({
          font: '20px sans-serif',
          fill: new Fill({
            color: 'rgba(255, 255, 255, 0.9)',
          }),
        }),
      }),
    });

    this.units = new VectorSource();
    const unitsVector = new VectorLayer({
      source: this.units,
      style: new Style({
        fill: new Fill({
          color: 'rgba(12, 190, 238, 0.1)',
        }),
        stroke: new Stroke({
          color: 'rgba(12, 190, 238, 0.9)',
          width: 3,
        }),
      }),
    });

    this.area = new VectorSource();
    const areaVector = new VectorLayer({
      source: this.area,
      style: new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.1)',
        }),
        stroke: new Stroke({
          color: 'rgba(204, 204, 204, 0.9)',
          width: 5,
        }),
      }),
    });

    this.map = new Map({
      target: 'map',
      layers: [raster, vector, unitsVector, areaVector],
      view: new View({
        projection: 'EPSG:3857',
        center: fromLonLat([lon, lat]),
        zoom: 17,
      }),
      interactions: defaults({ doubleClickZoom: false }),
    });

    const dragBox = new DragBox({
      condition: shiftKeyOnly,
    });
    this.map.addInteraction(dragBox);

    this.initCentralMarker(lon, lat);
    this.initFetch();

    dragBox.on('boxstart', () => {
      this.selectedUnits.forEach((f: any) => {
        f.setStyle(null);
      });
      this.selectedUnits = [];
    });
    dragBox.on('boxend', () => {
      const rotation = this.map.getView().getRotation();
      const extent = dragBox.getGeometry().getExtent();

      this.units.forEachFeatureIntersectingExtent(extent, (feature: any) => {
        if (feature.values_.move) {
          return;
        }

        this.selectedUnits.push(feature);
        feature.setStyle(
          new Style({
            stroke: new Stroke({
              color: 'rgb(115, 12, 238)',
              width: 3,
            }),
          })
        );
      });
    });
  }

  private async initFetch() {
    const currentExtent = this.map
      .getView()
      .calculateExtent(this.map.getSize());
    const TLpoint = getTopLeft(currentExtent);
    const TRpoint = getTopRight(currentExtent);
    const BRpoint = getBottomRight(currentExtent);
    const BLpoint = getBottomLeft(currentExtent);

    const TLcoords = transform(TLpoint, 'EPSG:3857', 'EPSG:4326');
    const TRcoords = transform(TRpoint, 'EPSG:3857', 'EPSG:4326');
    const BRcoords = transform(BRpoint, 'EPSG:3857', 'EPSG:4326');
    const BLcoords = transform(BLpoint, 'EPSG:3857', 'EPSG:4326');

    const normCoords = [BLcoords[1], BLcoords[0], TRcoords[1], TRcoords[0]];
    await this.fetchOverPass(normCoords);
  }

  private async fetchOverPass(normCoords: any[]) {
    const coords = `(${normCoords})`;
    const params = `[out:json][timeout:15];(node[building]${coords};way[building]${coords};relation[building]${coords};);out body geom;`;
    const baseUrl = `http://overpass-api.de/api/interpreter?data=${params}`;

    const { data } = await axios({
      method: 'get',
      url: baseUrl,
    });

    this.parseCoordinates(data);
  }

  private parseCoordinates(data: any): void {
    let { elements } = data;

    elements = elements.filter(({ geometry }: any) => {
      if (Array.isArray(geometry)) {
        return true;
      }
      return false;
    });

    const center = [];
    for (let i = 0; i < elements.length; i++) {
      const { geometry, id } = elements[i];

      const geometryCoordinates = [];
      for (let j = 0; j < geometry.length; j++) {
        const { lat, lon } = geometry[j];
        geometryCoordinates.push([lon, lat]);
      }

      const polygon = new Polygon([geometryCoordinates]);
      polygon.transform('EPSG:4326', 'EPSG:3857');

      const feature = new Feature({
        geometry: polygon,
        name: id,
        count: 0,
        max: 8,
        label: id,
      });

      this.source.addFeature(feature);
    }
    this.featureHoverListeners();
  }

  private featureHoverListeners(): void {
    this.map.on('click', e => {
      let insideCircle = false;
      this.map.forEachFeatureAtPixel(e.pixel, (f: any) => {
        const name = f.get('name');
        const olId = f.ol_uid;

        if (this.selectedUnits.length) {
          this.occupyFeature(olId, f);
          return;
        }

        if (name === 'circle') {
          insideCircle = true;
          return;
        }

        if (!insideCircle && !this.properties.length) {
          return;
        }

        if (!this.properties.length) {
          this.selectFeature(f);
        }
      });
    });
  }

  public selectFeature(f: any): void {
    const highlightStyle = new Style({
      fill: new Fill({
        color: 'rgba(255,255,255,0.7)',
      }),
      stroke: new Stroke({
        color: '#3399CC',
        width: 3,
      }),
    });

    const name = f.get('name');
    const olId = f.ol_uid;

    f.setStyle(highlightStyle);

    this.coins -= 10 * this.properties.length;
    this.properties.push(olId);

    this.propertiesCoordinates.push([f.getGeometry().getCoordinates()[0]]);

    if (this.areaFeature) {
      this.area.removeFeature(this.areaFeature);
    }

    const turfPoints: any[] = [];
    for (let i = 0; i < this.propertiesCoordinates.length; i++) {
      const property = this.propertiesCoordinates[i].forEach((coords: any) => {
        coords.forEach((coord: any) =>
          turfPoints.push(point(transform(coord, 'EPSG:3857', 'EPSG:4326')))
        );
      });
    }

    const convexCoorinates = convex(featureCollection(turfPoints));
    const coordinates: any[] = convexCoorinates.geometry.coordinates;

    const bufferCoordinates = transformScale(polygon(coordinates), 1.05)
      .geometry.coordinates;

    const polygon_ = new Polygon(bufferCoordinates);
    polygon_.transform('EPSG:4326', 'EPSG:3857');

    this.areaFeature = new Feature({
      geometry: polygon_,
    });

    this.area.addFeature(this.areaFeature);

    if (this.properties.length && this.globalCircle) {
      this.source.removeFeature(this.globalCircle);
      this.globalCircle = null;
    }
  }

  private async occupyFeature(id: any, f: any) {
    const sourceFeature: any = this.source.getFeatureByUid(id);
    if (!sourceFeature) {
      return;
    }
    const aimCoordinates = sourceFeature.getGeometry().getCoordinates();
    const aimCenter = centroid(polygon(aimCoordinates));
    const aimCenterCoordinate = aimCenter.geometry.coordinates;

    const completed: any[] = [];
    for (let i = 0; i < 1; i++) {
      completed[i] = false;
      const unit = this.selectedUnits[i];
      const olId = unit.ol_uid;

      const feature = this.units.getFeatureByUid(olId);
      feature.set('move', true);
      const parent = this.source.getFeatureByUid(feature.get('name'));

      const featureCoordinates = feature.getGeometry().getCenter();

      const range = [
        aimCenterCoordinate[0] - featureCoordinates[0],
        aimCenterCoordinate[1] - featureCoordinates[1],
      ];
      const rangeDivider = [
        Math.abs(Math.round(range[0] / 0.2)),
        Math.abs(Math.round(range[1] / 0.2)),
      ];

      let divideIteration = 0;
      let dividers: any[] = [];
      if (rangeDivider[0] <= rangeDivider[1]) {
        const divider = range[0] / rangeDivider[1];
        divideIteration = rangeDivider[1];
        dividers = [divider, 0.2];
      } else {
        const divider = range[1] / rangeDivider[0];
        divideIteration = rangeDivider[0];
        dividers = [0.2, divider];
      }

      console.log(range, rangeDivider, dividers, divideIteration);
      for (let i = 1; i < divideIteration; i++) {
        setTimeout(() => {
          feature
            .getGeometry()
            .setCenter([
              featureCoordinates[0] + dividers[0] * i,
              featureCoordinates[1] + dividers[1] * i,
            ]);
        }, 100 * i);
      }

      setTimeout(() => {
        feature.getGeometry().setCenter(aimCenterCoordinate);
        const count = parent.get('count') - 1;
        const lastChanged = parent.get('lastChanged');
        console.log(lastChanged);
        // parent.set('count', count);
        // this.units.removeFeature(feature);
        // completed[i] = true;
      }, 100 * divideIteration);
    }

    this.selectedUnits = [];

    return new Promise(resolve => {
      const intervalId = setInterval(() => {
        if (!completed.includes(false) && completed.length >= 8) {
          clearInterval(intervalId);
          this.selectFeature(f);
          resolve();
        }
      }, 100);
    });
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
@import '~ol/ol.css';
@import '~ol-ext/dist/ol-ext.css';

#map {
  height: 90vh;
  width: 90vw;
}
</style>
