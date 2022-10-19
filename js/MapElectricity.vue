<template>
  <!--换乘站-->
  <div className="main">
    <div id="map" style="width: 100%;height: 100vh;"></div>
  </div>
</template>
<script>
import $ from 'jquery'

let map
export default {
  name: "MapElectricity",
  props: {
    mapOptions: {
      type: Object,
      default: function () {
        return {
          zoom: 10.571875375656598,
          pitch: 0,
          bearing: -4.547473508864641e-13,
          center: [113.79646766138774, 34.64167422328043]
        }
      }
    }
  },
  //数据模型
  data() {
    return {}
  },
  //计算属性：类似data
  computed: {},
  //监控data数据变化
  watch: {},
  //创建前：没有数据模型和方法，没有html模板和渲染
  beforeCreate() {
  },
  //创建后：有数据模型和方法，没有html模板和渲染
  created() {
  },
  //挂载前：有数据模型和方法，没有html模板和渲染
  beforeMount() {
  },
  //挂载后：有数据模型和方法，有html模板和渲染
  mounted() {
    document.querySelector('body').setAttribute('class', 'main_body_item')

    this.mapInit()
    //-----------------------
    // $('body').on('click', function () {
    //   console.log('中心位置')
    //   console.log(map.getCenter())
    //   console.log('-------------------------------------------')
    //   console.log('x角度')
    //   console.log(map.getPitch())
    //   console.log('-------------------------------------------')
    //   console.log('y角度')
    //   console.log(map.getBearing())
    //   console.log('-------------------------------------------')
    //   console.log('放大倍数')
    //   console.log(map.getZoom())
    // })
  },
  //更新前：有数据模型和方法，有html模板，未渲染
  beforeUpdate() {
  },
  //更新后：有数据模型和方法，有html模板和渲染
  updated() {
  },
  //销毁前
  beforeDestroy() {
  },
  //销毁后
  destroyed() {
    document.querySelector('body').removeAttribute('class')
  },
  //如果页面有keep-alive缓存功能，这个函数会触发
  activated() {
  },
  //方法
  methods: {
    mapInit() {
      const baseMap = '../../data/map_electricity/zzarea.json'
      const baseMapAreaLine = '../../data/map_electricity/areaLine.json'
      const areaName = '../../data/map_electricity/areaName.json'
      const SubwayURL = './data/map/subway.json'
      const StationPtURL = './data/map/stationpoints.json'
      const SubstationData = './data/map_electricity/substation.json'
      const SubstationGLB = './data/map_electricity/zhus.fbx'
      const list = []
      const color = '#07609c'
      const colorMap = {
        "Line01": color,
        'Line02': color,
        'Line03': color,
        'Line04': color,
        "Line05": color,
        "Line06": color,
        "Line09": color,
        "Line10": color,
        "Line14": color,
        "Line17": color
      };
      map = new maptalks.Map("map", {
        center: this.mapOptions.center,
        zoom: this.mapOptions.zoom,
        minZoom: 10, // set map's min zoom to 14
        maxZoom: 20, // set map's max zoom to 14
        pitch: this.mapOptions.pitch,
        bearing: this.mapOptions.bearing,
        // dragRotate: false,
        //dragPitch : false,
        // centerCross: true,
        doubleClickZoom: false,
      });
      let threeLayer = new maptalks.ThreeLayer('t', {
        forceRenderOnMoving: true,
        forceRenderOnRotating: true,
      });
      let devLayer = new maptalks.VectorLayer('lineSt', {
        altitude: 2022
      })
      let vector = new maptalks.VectorLayer('vector', {
        altitude: 2022,
      })
      threeLayer.prepareToDraw = function (gl, scene) {
        let light = new THREE.DirectionalLight(0xffffff, 1);
        light.position.set(0, -10, 10).normalize();
        scene.add(light);
        scene.add(new THREE.AmbientLight('#fff', 1));
        /*----------------------------------------------------*/
        //外框呼吸闪烁效果
        threeLayer.initOutLinePass()
        /*----------------------------------------------------*/
        //加载变电所头部信息
        loadSubstationHeadInfo()
        //加载地图线
        loadSubway(baseMapAreaLine, 2020, 1)
        //加载结构
        loadStructure(baseMap, 'one', 2000, 20)
        loadAreaName()//加区域名称
        loadMetroLine(SubwayURL)//加载地铁线路 主线
        createLineMarker()//加载站点
        loadStation()//加站点
        loadSubstationName()//加载变电所

        animate();
      };
      threeLayer.addTo(map);
      devLayer.addTo(map);
      vector.addTo(map);

      var ballElectric, material;

      //添加笼罩
      function addElectricShield(g) {
        material = getMaterials();
        ballElectric = new ElectricShield(g.geometry.coordinates, {
          radius: g.properties.radius,
          altitude: 2022
        }, material, threeLayer)
        threeLayer.addMesh(ballElectric);
      }

      let stationInfo = []
      let stationR = 25

      function loadStation() {
        fetch(StationPtURL).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (stationData) {
          let material = new THREE.MeshBasicMaterial({color: '#fff', opacity: 1, transparent: true});//#3e35cf//5c57b8//#fad02c
          let highlightmaterial = new THREE.MeshBasicMaterial({color: 'yellow', transparent: true});

          let heightPerLevel = 1;
          let features = stationData.features;
          features.forEach(function (g) {
            let levels = 2//g.properties.Floor || 1;
            let devCrd = g.geometry.coordinates

            let circle = new maptalks.Circle(devCrd, stationR, {'name': g.properties.Name,});
            let mesh = threeLayer.toExtrudePolygon(circle, {
              height: heightPerLevel * levels,
              // width: 20,
              topColor: '#fff',
              altitude: 2050,
              name: g.properties.Name,
              line: g.properties.Line,
              stcord: devCrd,
            }, material);
            stationInfo.push(mesh)
          });
          //threeLayer.addMesh(stationInfo);
          if (threeLayer.delayAddMesh) {
            threeLayer.delayAddMesh(stationInfo);
          } else {
            threeLayer.addMesh(stationInfo);
          }
        });
      }

      //加载变电所头部信息
      function loadSubstationHeadInfo() {
        let data = []
        fetch(SubstationData).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (Substation) {
          let features = Substation.features;
          features.forEach(function (g) {
            let obj = {}
            obj['text'] = g.properties.name
            obj['color'] = g.properties.color
            obj['coordinate'] = g.geometry.coordinates
            data.push(obj)
          });
          data.forEach(element => {
            const style = {
              // x: width * Math.random(),
              // y: height * Math.random(),
              text: element.text,
              // width: 50,
              height: 50,
              textFill: element.color,
              // scale fontSize 18*2
              textFont: '36px Microsoft Yahei',
              // textBackgroundColor: null,
              // textPadding: [10, 15],
              textShadowColor: '#fff',
              textShadowBlur: 2
            };
            const text = new zrender.Text({
              style
            });
            // zr.add(text);
            const rect = getRect(text.getBoundingRect());
            createZr(element, rect, style, function () {
              if (list.length === data.length) {
                addSprites();
              }
            });
          });
        });
      }

      // function animation() {
      //   // layer animation support Skipping frames
      //   threeLayer._needsUpdate = !threeLayer._needsUpdate;
      //   if (threeLayer._needsUpdate && !threeLayer.isRendering()) {
      //     threeLayer.redraw();
      //   }
      //   /*------------------------刷新--------------------------*/
      //   //外框呼吸闪烁效果
      //   threeLayer.composer.render(1)
      //   /*----------------------------------------------------*/
      //   threeLayer.loop && threeLayer.loop(false);
      //   requestAnimationFrame(animation);
      // }

      function animation() {
        // layer animation support Skipping frames
        threeLayer._needsUpdate = !threeLayer._needsUpdate;
        if (threeLayer._needsUpdate) {
          threeLayer.renderScene();
          /*------------------------刷新--------------------------*/
          //外框呼吸闪烁效果
          threeLayer.composer.render(1)
          /*----------------------------------------------------*/
        }
        threeLayer.loop && threeLayer.loop(false);
        // stats.update();
        requestAnimationFrame(animation);
      }

      //生成线站名标识
      function createLineMarker() {
        fetch(StationPtURL).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text).features;
        }).then(function (features) {
          features.map(feature => {
            createStMarker(feature, 'drw').addTo(devLayer)
          })
        })
      }

      //缩放事件站名按类型显示
      map.on('zooming', function (param) {
        let zm = map.getZoom()
        devLayer._geoList.forEach(function (dev) {
          setStationNameShow(zm, dev)
        })
      })

      //生成单站站名标识
      function createStMarker(feature) {
        let devCrd = feature.geometry.coordinates
        let marker = new maptalks.Marker(
            devCrd,
            {
              'properties': {
                'name': feature.properties.Name,
                'shadowColor': '#00f',
                'line': feature.properties.Line,
                'type': feature.properties.Type,
                'cordX': devCrd,
              },
              'symbol': [
                {
                  'textName': '{name}',
                  'textSize': 5,
                  'textDy': -15,
                  'textFill': '#fff',
                  'textWeight': 'normal', //'bold', 'bolder'
                }
              ]
            }
        )
        let zm = map.getZoom()
        setStationNameShow(zm, marker)
        return marker
      }

      //缩放站场名显示过滤
      function setStationNameShow(zm, dev) {
        if (zm <= 11) {
          setTextShow(dev, 0)
        } else if (zm > 11 && zm < 12) {
          if (dev.properties.type === 2) {
            setTextShow(dev, 1)
          } else {
            setTextShow(dev, 0)
          }
        } else if (zm >= 12 && zm <= 12.5) {
          if (dev.properties.type === 1) {
            setTextShow(dev, 0)
          } else {
            setTextShow(dev, 1)
          }
        } else {
          setTextShow(dev, 1)
        }
      }

      function setTextShow(dev, num) {
        dev.setSymbol({
          'textName': dev.properties.name,
          'textSize': 10,
          'textDy': -20,
          // 'textColor': '#fff',
          'textFill': '#008cb6',
          'textWeight': 'normal',
          'textOpacity': num,
        })
      }

      function loadMetroLine(SubwayURL) {
        fetch(SubwayURL).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (json) {
          var features = json.features;
          features.forEach(function (geojson) {
            drawMetroLine(geojson)
          })
        })
      }

      let subwayInfo = []

      function drawMetroLine(geojson) {
        let lineString = maptalks.GeoJSON.toGeometry(geojson);
        let lineShow = lineString.properties.ShowName;
        let material = new THREE.MeshBasicMaterial(
            {
              color: colorMap[lineString.properties.Name],
              opacity: 0.4,
              transparent: true
            });
        let line = threeLayer.toExtrudeLine(lineString,
            {
              altitude: 2022,
              width: 80,
              height: 2
            },
            material);
        line.setProperties({
          'cordX': line.getObject3d().position.x
        })
        line.setToolTip(lineShow, {
          showTimeout: 0,
          eventsPropagation: true,
          dx: 10,
        });
        subwayInfo.push(line)
        threeLayer.addMesh(line);
      }

      var groundWall, material;

      function addWall(arr, height) {
        material = getMaterial();
        groundWall = new RippleWall(
            arr,
            {height: height},
            material,
            threeLayer)
        threeLayer.addMesh(groundWall);
      }

      function getMaterial() {
        const vertexs = {
          normal_vertex: "\n  precision lowp float;\n  precision lowp int;\n  ".concat(THREE.ShaderChunk.fog_pars_vertex, "\n  varying vec2 vUv;\n  void main() {\n    vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );\n    vUv = uv;\n    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);\n    ").concat(THREE.ShaderChunk.fog_vertex, "\n  }\n"),
        }

        const fragments = {
          rippleWall_fragment: "\n  precision lowp float;\n  precision lowp int;\n  uniform float time;\n  uniform float opacity;\n  uniform vec3 color;\n  uniform float num;\n  uniform float hiz;\n\n  varying vec2 vUv;\n\n  void main() {\n    vec4 fragColor = vec4(0.);\n    float sin = sin((vUv.y - time * hiz) * 10. * num);\n    float high = 0.92;\n    float medium = 0.4;\n    if (sin > high) {\n      fragColor = vec4(mix(vec3(.8, 1., 1.), color, (1. - sin) / (1. - high)), 1.);\n    } else if(sin > medium) {\n      fragColor = vec4(color, mix(1., 0., 1.-(sin - medium) / (high - medium)));\n    } else {\n      fragColor = vec4(color, 0.);\n    }\n\n    vec3 fade = mix(color, vec3(0., 0., 0.), vUv.y);\n    fragColor = mix(fragColor, vec4(fade, 1.), 0.85);\n    gl_FragColor = vec4(fragColor.rgb, fragColor.a * opacity * (1. - vUv.y));\n  }\n",
        }
        const material = new THREE.ShaderMaterial({
          uniforms: {
            time: {
              type: "pv2",
              value: 0
            },
            color: {
              type: "uvs",
              value: new THREE.Color('#409291')
            },
            opacity: {
              type: "pv2",
              value: 0.8
            },
            num: {
              type: "pv2",
              value: 5
            },
            hiz: {
              type: "pv2",
              value: 0.15
            }
          },
          vertexShader: vertexs.normal_vertex,
          fragmentShader: fragments.rippleWall_fragment,
          blending: THREE.AdditiveBlending,
          transparent: !0,
          depthWrite: !1,
          depthTest: !0,
          side: THREE.DoubleSide
        });
        return material;
      }

      function loadAreaName() {
        fetch(areaName).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (areaNameData) {
          let features = areaNameData.features;
          features.forEach(function (g) {
            var point = new maptalks.Marker(
                g.geometry.coordinates,
                {
                  'properties': {
                    'name': g.properties.name
                  },
                  'symbol': {
                    'textFaceName': 'sans-serif',
                    'textName': '{name}',          //value from name in geometry's properties
                    'textSize': 16,
                    'textFill': '#72F7F9',
                    'textOpacity': 0.3,
                  }
                }
            );
            point.addTo(vector)
          });
        });
      }

      function loadSubstationName() {
        fetch(SubstationData).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (Substation) {
          let features = Substation.features;
          features.forEach(function (g) {
            setGlb(g)
            //添加笼罩
            //addElectricShield(g)
            //加载变电所名称
            //biandiansuoName(g)
          });
        });
      }

      //加载变电所名称
      function biandiansuoName() {
        fetch(SubstationData).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (areaNameData) {
          let features = areaNameData.features;
          features.forEach(function (g) {
            var point = new maptalks.Marker(
                g.geometry.coordinates,
                {
                  'properties': {
                    'name': g.properties.name
                  },
                  'symbol': {
                    'textFaceName': 'sans-serif',
                    'textName': '{name}',          //value from name in geometry's properties
                    'textSize': 16,
                    'textFill': '#72F7F9',
                    'textOpacity': 0.3,
                  }
                }
            );
            point.addTo(vector)
          });
        });
      }

      //设置glb
      let trainModelList = []

      function setGlb(data) {
        // const loader = new THREE.GLTFLoader();
        // // 实例化加载器
        // loader.load(SubstationGLB, (gltf) => {
        //   let model = gltf.scene;
        //   let set = 0.0002
        //   let altitude = 2022
        //   let devAngle = 0
        //   model.rotation.x = Math.PI * 1 / 2;
        //   model.rotation.y = devAngle;
        //   model.scale.set(set, set, set);
        //   let z = threeLayer.distanceToVector3(altitude, altitude).x;
        //   model.position.copy(threeLayer.coordinateToVector3(data.geometry.coordinates, z));
        //   threeLayer.addMesh(model)
        // });
        var loader = new THREE.FBXLoader();
        loader.load(SubstationGLB, function (fbx) {
          fbx.scale.set(0.0002, 0.0002, 0.0002);
          fbx.rotation.set(Math.PI * 1 / 2, 0, 0);
          /*----------------------------------------------------*/
          //外框呼吸闪烁效果
          let childMesh = []
          fbx.traverse(child => {
            if (child.isMesh && child.geometry.isBufferGeometry) {
              childMesh.push((child))
              trainModelList.push(child)
            }
          })

          /*----------------------------------------------------*/

          let model = threeLayer.toModel(fbx, {
            coordinate: data.geometry.coordinates,
            altitude: 2022,
          });
          //外框呼吸闪烁效果
          threeLayer.outlinePass.selectedObjects = []
          threeLayer.outlinePass.selectedObjects = trainModelList
          threeLayer.delayAddMesh(model);
        })
      }

      //加载结构
      function loadStructure(subwayURL, type, altitude_number, height) {
        fetch(subwayURL).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (json) {
          var features = json.features;
          features.forEach(function (geojson) {
            //结构设置
            structureSet(geojson, type, altitude_number, height)
          })
        })
      }

      let secListInfo = []

      function loadSubway(subwayURL, altitude_height, height) {
        fetch(subwayURL).then(function (res) {
          return res.text();
        }).then(function (text) {
          return JSON.parse(text);
        }).then(function (json) {
          var features = json.features;
          features.forEach(function (geojson) {
            if (geojson.properties.id === 29) {
              addWall(geojson.geometry.coordinates, 2000)
            } else {
              borderLine(geojson, altitude_height, height)
            }
          })
          if (threeLayer.delayAddMesh) {
            threeLayer.delayAddMesh(secListInfo);
          } else {
            threeLayer.addMesh(secListInfo);
          }
        })
      }

      function borderLine(geojson, altitude_height, height) {
        let lineString = maptalks.GeoJSON.toGeometry(geojson);
        let material = new THREE.MeshBasicMaterial(
            {
              color: '#72F7F9',
              opacity: 1,
              transparent: true
            });
        let line = threeLayer.toExtrudeLine(lineString, {
          altitude: altitude_height,
          width: 15,
          height: height
        }, material);
        threeLayer.addMesh(line);
      }

      // $('body').on('click', function () {
      //   console.log('中心位置')
      //   console.log(map.getCenter())
      //   console.log('-------------------------------------------')
      //   console.log('x角度')
      //   console.log(map.getPitch())
      //   console.log('-------------------------------------------')
      //   console.log('y角度')
      //   console.log(map.getBearing())
      //   console.log('-------------------------------------------')
      //   console.log('放大倍数')
      //   console.log(map.getZoom())
      // })

      //结构设置
      function structureSet(f, type, altitude_number, h) {
        //材料颜色设置
        let material = new THREE.MeshLambertMaterial({color: '#409291', transparent: true, opacity: 0.3});
        let altitude = altitude_number
        let height = h
        let topColor = '#007697'
        let bottomColor = '#00485a'
        const extrudePolygon = threeLayer.toExtrudePolygon(f, {
          topColor: topColor,
          bottomColor: bottomColor,
          altitude: altitude,
          height: height,
          interactive: true,
        }, material);
        extrudePolygon.off('click', function () {
          //console.log('点击事件')
        })
        threeLayer.addMesh(extrudePolygon);
      }

      function animate() {
        threeLayer._needsUpdate = !threeLayer._needsUpdate;
        if (threeLayer._needsUpdate) {
          threeLayer.renderScene();
        }
        threeLayer.loop && threeLayer.loop(false);
        requestAnimationFrame(animation);
      }

      function getMaterials() {
        var ElectricShield = {
          uniforms: {
            time: {
              type: "f",
              value: 0
            },
            color: {
              type: "c",
              value: new THREE.Color("#71f6f8")
            },
            opacity: {
              type: "f",
              value: 1
            }
          },
          vertexShaderSource: "\n  precision lowp float;\n  precision lowp int;\n  "
              .concat(
                  THREE.ShaderChunk.fog_pars_vertex,
                  "\n  varying vec2 vUv;\n  void main() {\n    vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );\n    vUv = uv;\n    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);\n    "
              )
              .concat(THREE.ShaderChunk.fog_vertex, "\n  }\n"),
          fragmentShaderSource: `
                #if __VERSION__ == 100
                 #extension GL_OES_standard_derivatives : enable
                #endif
                uniform vec3 color;
                uniform float opacity;
                uniform float time;
                varying vec2 vUv;
                #define pi 3.1415926535
                #define PI2RAD 0.01745329252
                #define TWO_PI (2. * PI)
                float rands(float p){
                    return fract(sin(p) * 10000.0);
                }
                float noise(vec2 p){
                    float t = time / 20000.0;
                    if(t > 1.0) t -= floor(t);
                    return rands(p.x * 14. + p.y * sin(t) * 0.5);
                }
                vec2 sw(vec2 p){
                    return vec2(floor(p.x), floor(p.y));
                }
                vec2 se(vec2 p){
                    return vec2(ceil(p.x), floor(p.y));
                }
                vec2 nw(vec2 p){
                    return vec2(floor(p.x), ceil(p.y));
                }
                vec2 ne(vec2 p){
                    return vec2(ceil(p.x), ceil(p.y));
                }
                float smoothNoise(vec2 p){
                    vec2 inter = smoothstep(0.0, 1.0, fract(p));
                    float s = mix(noise(sw(p)), noise(se(p)), inter.x);
                    float n = mix(noise(nw(p)), noise(ne(p)), inter.x);
                    return mix(s, n, inter.y);
                }
                float fbm(vec2 p){
                    float z = 2.0;
                    float rz = 0.0;
                    vec2 bp = p;
                    for(float i = 1.0; i < 6.0; i++){
                    rz += abs((smoothNoise(p) - 0.5)* 2.0) / z;
                    z *= 2.0;
                    p *= 2.0;
                    }
                    return rz;
                }
                void main() {
                    vec2 uv = vUv;
                    vec2 uv2 = vUv;
                    if (uv.y < 0.5) {
                    discard;
                    }
                    uv *= 4.;
                    float rz = fbm(uv);
                    uv /= exp(mod(time * 2.0, pi));
                    rz *= pow(15., 0.9);
                    gl_FragColor = mix(vec4(color, opacity) / rz, vec4(color, 0.1), 0.2);
                    if (uv2.x < 0.05) {
                    gl_FragColor = mix(vec4(color, 0.1), gl_FragColor, uv2.x / 0.05);
                    }
                    if (uv2.x > 0.95){
                    gl_FragColor = mix(gl_FragColor, vec4(color, 0.1), (uv2.x - 0.95) / 0.05);
                    }
                }`
        };
        let material = new THREE.ShaderMaterial({
          uniforms: ElectricShield.uniforms,
          defaultAttributeValues: {},
          vertexShader: ElectricShield.vertexShaderSource,
          fragmentShader: ElectricShield.fragmentShaderSource,
          blending: THREE.AdditiveBlending,
          depthWrite: !1,
          depthTest: !0,
          side: THREE.DoubleSide,
          transparent: !0
        });
        return material;
      }

      var OPTIONS = {
        altitude: 0,
        speed: 0.015,
        height: 10
      };

      class RippleWall extends maptalks.BaseObject {
        constructor(polygon, options, material, layer) {
          if (Array.isArray(polygon)) {
            polygon = new maptalks.Polygon([polygon]);
          }
          options = maptalks.Util.extend({}, OPTIONS, options, {layer, polygon});
          super();
          this._initOptions(options);
          const {altitude, height} = options;
          const geometry = this.createGeometry(polygon, layer, height)
          this._createMesh(geometry, material);
          const z = layer.distanceToVector3(altitude, altitude).x;
          const position = layer.coordinateToVector3(polygon.getCenter(), z);
          this.getObject3d().position.copy(position);
        }

        createGeometry(polygon, layer, height) {
          height = layer.distanceToVector3(height, height).x;
          const centerPt = layer.coordinateToVector3(polygon.getCenter());
          const wall = polygon.getShell();
          const positionsV = [];
          let joinLonLat = [];
          wall.forEach(lnglat => {
            const polyPice = layer.coordinateToVector3(lnglat).sub(centerPt);
            positionsV.push(polyPice);
            joinLonLat.push(polyPice.x);
            joinLonLat.push(polyPice.y);
          });
          for (var a = joinLonLat, polySub = [], o = 0, s = 0; o < a.length - 2; o += 2, s++)
            0 === o ?
                polySub[0] = Math.sqrt((a[2] - a[0]) * (a[2] - a[0]) + (a[3] - a[1]) * (a[3] - a[1])) :
                polySub[s] = polySub[s - 1] + Math.sqrt((a[o + 2] - a[o]) * (a[o + 2] - a[o]) + (a[o + 3] - a[o + 1]) * (a[o + 3] - a[o + 1]));
          let pos = [],
              uvs = [];
          let polylenth = polySub[polySub.length - 1];
          for (let d = 0, u = pos.length, p = uvs.length; d < positionsV.length - 1; d++) {
            let pv1 = positionsV[d],
                pv2 = positionsV[d + 1],
                polyPice = polySub[d];
            pos[u++] = pv1.x,
                pos[u++] = pv1.y,
                pos[u++] = 0,
                uvs[p++] = 0 === d ? 0 : polySub[d - 1] / polylenth,
                uvs[p++] = 0,
                pos[u++] = pv2.x,
                pos[u++] = pv2.y,
                pos[u++] = 0,
                uvs[p++] = polyPice / polylenth,
                uvs[p++] = 0,
                pos[u++] = pv1.x,
                pos[u++] = pv1.y,
                pos[u++] = height,
                uvs[p++] = 0 === d ? 0 : polySub[d - 1] / polylenth,
                uvs[p++] = 1,
                pos[u++] = pv1.x,
                pos[u++] = pv1.y,
                pos[u++] = height,
                uvs[p++] = 0 === d ? 0 : polySub[d - 1] / polylenth,
                uvs[p++] = 1,
                pos[u++] = pv2.x,
                pos[u++] = pv2.y,
                pos[u++] = 0,
                uvs[p++] = polyPice / polylenth,
                uvs[p++] = 0,
                pos[u++] = pv2.x,
                pos[u++] = pv2.y,
                pos[u++] = height,
                uvs[p++] = polyPice / polylenth,
                uvs[p++] = 1
          }
          var geometry = new THREE.BufferGeometry;
          geometry.setAttribute("position", new THREE.BufferAttribute(new Float32Array(pos), 3));
          geometry.setAttribute("uv", new THREE.BufferAttribute(new Float32Array(uvs), 2));
          return geometry;
        }


        _animation() {
          const wall = this.getObject3d();
          const speed = this.getOptions().speed;
          wall.material.uniforms.time.value += speed;
        }
      }

      class ElectricShield extends maptalks.BaseObject {
        constructor(coordinate, options, material, layer) {
          options = maptalks.Util.extend({}, OPTIONS, options, {layer, coordinate});
          super();
          this._initOptions(options);
          const {altitude, radius} = options;
          const r = layer.distanceToVector3(radius, radius).x;
          const geometry = new THREE.SphereBufferGeometry(r, 50, 50, 0, Math.PI * 2);
          this._createGroup();
          const mesh = new THREE.Mesh(geometry, material);
          this.getObject3d().add(mesh);
          const z = layer.distanceToVector3(altitude, altitude).x;
          const position = layer.coordinateToVector3(coordinate, z);
          this.getObject3d().position.copy(position);
          this.getObject3d().rotation.x = Math.PI / 2;
        }

        _animation() {
          const ball = this.getObject3d().children[0];
          const speed = this.getOptions().speed;
          ball.material.uniforms.time.value += speed;
        }
      }

      //加载变电所头部信息
      function getRect(bound) {
        const {width, height, x, y} = bound;
        const w = Math.max(2, THREE.Math.ceilPowerOfTwo(width));
        const h = Math.max(2, THREE.Math.ceilPowerOfTwo(height));
        return {
          width, height, w, h
        };
      }

      function createZr(d, rect, style, callback) {
        const {w, h, width, height} = rect;
        let canvas = document.createElement('canvas');
        canvas.width = w;
        canvas.height = h;
        const zr = zrender.init(canvas, {
          width: w,
          height: h
        });
        const options = Object.assign({}, style);
        options.x = (w / 2 - width / 2);
        options.y = (h / 2 - height / 2);
        const text = new zrender.Text({
          style: options
        });
        zr.add(text);
        zr.on('rendered', () => {
          d.image = canvas.toDataURL();
          d.rect = rect;
          list.push(d);
          callback();
          zr.dispose();
          canvas = null;
        })
      }

      var textSprites = [];
      const textureLoader = new THREE.TextureLoader();

      function addSprites() {
        list.forEach(d => {
          const {coordinate, image, color, rect} = d;
          const material = new THREE.SpriteMaterial({map: textureLoader.load(image)});
          material.needsUpdate = true;
          const alt = Math.random() * 50 + 7022;
          const text = new TextSprite(coordinate, {altitude: 9500, rect}, material, threeLayer);
          const line = new Line(coordinate, {height: alt, interactive: false}, new THREE.LineBasicMaterial({
            color,
            transparent: true,
            // opacity: 0.3
          }), threeLayer);
          textSprites.push(line, text);
        });
        threeLayer.addMesh(textSprites);
        animation();
      }

      class TextSprite extends maptalks.BaseObject {
        constructor(coordinate, options, material, layer) {
          options = maptalks.Util.extend({}, OPTIONS, options, {layer, coordinate});
          super();
          this._initOptions(options);
          const {altitude, rect} = options;
          const {w, h} = rect;
          this._createGroup();
          const textsprite = new THREE.Sprite(material);
          textsprite.material.sizeAttenuation = false;
          const scale = 0.01 / 10 / 3;
          textsprite.scale.set(scale * w, scale * h, 1);
          this.getObject3d().add(textsprite);
          const z = layer.distanceToVector3(altitude, altitude).x;
          const position = layer.coordinateToVector3(coordinate, z);
          this.getObject3d().position.copy(position);
          this._vector = new THREE.Vector3();
          this._rect = rect;
        }

        getTextRect() {
          this.getPixel();
          const {x, y} = this._pixel;
          const {width, height} = this._rect;
          return {
            minX: x - width / 4,
            minY: y - height / 4,
            maxX: x + width / 4,
            maxY: y + height / 4
          }
        }

        getPixel() {
          const size = this.getMap().getSize();
          const camera = this.getLayer().getCamera();
          const position = this.getObject3d().position;
          this._vector.x = position.x;
          this._vector.y = position.y;
          this._vector.z = position.z;
          this._pixel = simplepath.vector2Pixel(this._vector, size, camera);
        }

        identify(coordinate) {
          const {minX, minY, maxX, maxY} = this.getTextRect();
          const pixel = this.getMap().coordToContainerPoint(coordinate);
          if (pixel.x >= minX && pixel.x <= maxX && pixel.y >= minY && pixel.y <= maxY) {
            return true;
          }
          return false;
        }
      }

      var OPTIONS1 = {
        altitude: 2022,
        height: 10
      }

      class Line extends maptalks.BaseObject {
        constructor(coordinate, options, material, layer) {
          options = maptalks.Util.extend({}, OPTIONS1, options, {layer, coordinate});
          super();
          this._initOptions(options);
          const {altitude, height} = options;
          const h = layer.distanceToVector3(height, height).x;
          const geometry = new THREE.BufferGeometry();
          const positions = [0, 0, 0, 0, 0, h];
          geometry.addAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
          this._createLine(geometry, material);
          const z = layer.distanceToVector3(altitude, altitude).x;
          const position = layer.coordinateToVector3(coordinate, z);
          this.getObject3d().position.copy(position);
        }
      }

      var simplepath = {
        positionsConvert: function (worldPoints, altitude = 0, layer) {
          const vectors = [];
          for (let i = 0, len = worldPoints.length; i < len; i += 3) {
            let x = worldPoints[i], y = worldPoints[i + 1], z = worldPoints[i + 2];
            if (altitude > 0) {
              z += layer.distanceToVector3(altitude, altitude).x;
            }
            vectors.push(new THREE.Vector3(x, y, z));
          }
          return vectors;
        },
        vectors2Pixel: function (worldPoints, size, camera, altitude = 0, layer) {
          if (!(worldPoints[0] instanceof THREE.Vector3)) {
            worldPoints = simplepath.positionsConvert(worldPoints, altitude, layer);
          }
          const pixels = worldPoints.map(worldPoint => {
            return simplepath.vector2Pixel(worldPoint, size, camera);
          })
          return pixels;
        },
        vector2Pixel: function (world_vector, size, camera) {
          const vector = world_vector.project(camera);
          const halfWidth = size.width / 2;
          const halfHeight = size.height / 2;
          const result = {
            x: Math.round(vector.x * halfWidth + halfWidth),
            y: Math.round(-vector.y * halfHeight + halfHeight)
          };
          return result;
        },
      };
      //
      /*----------------------------------------------------*/
      //外轮廓高亮显示 外框呼吸闪烁效果
      maptalks.ThreeLayer.prototype.initOutLinePass = function () {
        const renderer = this.getThreeRenderer();
        const size = this.getMap().getSize();
        this.composer = new THREE.EffectComposer(renderer);
        this.composer.setSize(size.width, size.height);
        const scene = this.getScene(), camera = this.getCamera();
        this.renderPass = new THREE.RenderPass(scene, camera);
        this.composer.addPass(this.renderPass);

        const outlinePass = this.outlinePass = new THREE.OutlinePass(new THREE.Vector2(size.width, size.height), scene, camera);
        //定义外框全局变量1
        outlinePass.edgeStrength = 3;//粗
        outlinePass.edgeGlow = 4;//发光
        outlinePass.edgeThickness = 2;//光晕粗
        outlinePass.pulsePeriod = 0;//闪烁
        outlinePass.usePatternTexture = false;//是否使用贴图
        outlinePass.visibleEdgeColor.set('#f00'); // 设置显示的颜色
        outlinePass.hiddenEdgeColor.set('white'); // 设置隐藏的颜色
        outlinePass.renderToScreen = true
        this.composer.addPass(outlinePass);

        //定义外框全局变量2
        const outlinePass1 = this.outlinePass1 = new THREE.OutlinePass(new THREE.Vector2(size.width, size.height), scene, camera);
        outlinePass1.edgeStrength = 3;//粗
        outlinePass1.edgeGlow = 3;//发光
        //outlinePass1.edgeThickness = 2;//光晕粗
        //outlinePass1.pulsePeriod = 2;//闪烁
        outlinePass1.usePatternTexture = false;//是否使用贴图
        outlinePass1.visibleEdgeColor.set('green'); // 设置显示的颜色
        outlinePass1.hiddenEdgeColor.set('white'); // 设置隐藏的颜色
        outlinePass1.renderToScreen = true
        this.composer.addPass(outlinePass1);

        this.outlineEnable = true
      }
      /*----------------------------------------------------*/
    }
  },
}
</script>

<style scoped>
</style>