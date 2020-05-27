<template>
    <div>
        <div class="columns is-centered"
             style="position:relative">
            <b-loading :active.sync="loading"
                       :is-full-page="false"
                       relative></b-loading>
            <canvas id="canvas"
                    :width="width"
                    :height="height"
                    ref="canvas"></canvas>
        </div>
        <br>
        <div class="columns">
            <div class="column">
                <div class="container-align-middle">
                    <div>
                        <h2 class="has-margin-bottom-md">
                            <i class="fas fa-paint-brush"></i> Brush Type
                        </h2>
                        <div v-for="(color,key) in colors"
                             :key="key">
                            <div class="control">
                                <div class="field set-cursor"
                                     @click="activeColorKey = key">
                                    <div :style="{'background-color' : color}"
                                         class="color-button"
                                         :class="{'active': activeColorKey == key}"></div>
                                    <div class="display-key"
                                         :class="{'active': activeColorKey == key}">
                                        {{_.upperFirst(key)}}
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
            <div class="column">
                <div class="container-align-middle">
                    <div>
                        <h2 class="has-margin-bottom-md">
                            <i class="fas fa-circle"></i> Brush Size
                        </h2>
                        <div>
                            <button v-for="size in sizes"
                                    :key="size"
                                    class="size-button button has-margin-right-sm"
                                    @click="activeSize = size"
                                    :class="{'active': activeSize == size}">{{ size }} px
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <footer class="modal-card-foot">
            <div class="field is-grouped">
                <div class="control">
                    <button @click="undo()"
                            class="button"
                            :disabled="!canUndo">Undo
                    </button>
                </div>
                <div class="control">
                    <button class="button"
                            @click="reset()"
                            :disabled="!canUndo">Reset All
                    </button>
                </div>
                <div class="control">
                    <button @click="createExport()"
                            class="button is-primary">Upload
                    </button>
                </div>
            </div>
        </footer>
    </div>
</template>

<script>
    import paper from "paper";
    import Vue from 'vue';
    import _ from 'lodash';
    import Buefy from 'buefy';
    import 'buefy/dist/buefy.css';

    Vue.use(Buefy, {
        defaultIconPack: 'fas',
    });

    Vue.prototype._ = _;

    export default {
        props: {
            src: {
                type: String,
                required: true
            },
            colors: {
                type: Object,
                default() {
                    return {
                        Keep: "rgba(0,255,0,0.5)",
                        Disregard: "rgba(255,0,0,0.5)",
                        "Keep as 2D": "rgba(0, 0, 255,0.5)"
                    };
                }
            },
            color: {default: "Keep"},
            sizes: {
                type: Array,
                default() {
                    return [20, 40, 70];
                }
            },
            size: {
                default: 40
            },
            openResult: {
                type: Boolean,
                default: false
            }
        },
        async mounted() {
            console.log("Mounted");
            this.addImage().then(this.startDrawing);
        },
        data() {
            return {
                loading: true,
                width: 600,
                height: 500,
                paperScope: new paper.PaperScope(),
                baseRaster: null,
                activePath: null,
                activeColorKey: this.color,
                activeSize: this.size,
                lastExport: null,
                paths: []
            };
        },
        computed: {
            tool() {
                return _.get(this.paperScope, "tool", new this.paperScope.Tool());
            },
            activeColor() {
                return this.colors[this.activeColorKey];
            },
            canUndo() {
                return _.get(this.paths, "length") > 0;
            }
        },
        watch: {
            canUndo() {
                this.$emit("can-undo-changed", this.canUndo);
            }
        },
        methods: {
            async addImage(src = null) {
                src = src ? src : this.src;
                console.log("Add image");
                this.paperScope.setup(this.$refs.canvas);
                this.paperScope.project.options.handleSize = 15;
                let position = this.paperScope.view.center;

                /* possible cors fix */
                /*  await new Promise(resolve => {
                      let img = new Image();
                      img.crossOrigin = "Anonymous";
                      img.addEventListener("load", resolve, false);
                      img.src = src;
                  });*/

                return new Promise((resolve, reject) => {
                    let originalRaster = new this.paperScope.Raster({
                        crossOrigin: "anonymous",
                        source: src,
                        position: position
                    });
                    originalRaster.onLoad = image => {
                        this.baseRaster = originalRaster;
                        this.baseRaster.fitBounds(this.paperScope.view.bounds);
                        this.baseRaster = originalRaster.rasterize(150);
                        originalRaster.remove();
                        resolve(image);
                    };
                    originalRaster.onError = () => {
                        reject(new Error(`Failed to load image from: ${src}`));
                    };
                });
            },
            startDrawing() {
                console.log("Start Drawing");
                this.tool.minDistance = 1;
                this.loading = false;
                this.tool.onMouseDown = event => {
                    this.activePath = new paper.Path();
                    this.activePath.strokeColor = this.activeColor;
                    this.activePath.strokeWidth = this.activeSize;
                    this.activePath.strokeCap = "round";
                    this.activePath.strokeJoin = "round";
                    this.activePath.blendMode = "source-atop";
                    this.activePath.add(event.point);
                };

                this.tool.onMouseDrag = event => {
                    this.activePath.add(event.point);
                };

                this.tool.onMouseUp = event => {
                    this.activePath.add(event.point);
                    this.paths.push(this.activePath);
                };
            },
            undo() {
                if (this.canUndo) {
                    this.paperScope.project.activeLayer.removeChildren(this.paths.length);
                    this.paths.pop();
                    this.paperScope.view.draw();
                }
            },
            reset() {
                this.paths = [];
                this.paperScope.project.activeLayer.removeChildren(1); //TODO the first is image, need to check types here
                this.paperScope.view.draw();
            },
            async createExport() {
                let clippingBox = new this.paperScope.Path.Rectangle(
                    this.baseRaster.bounds
                );
                let clippingGroup = new this.paperScope.Group(
                    clippingBox,
                    this.baseRaster,
                    ...this.paths
                );
                clippingGroup.clipped = true;
                let raster = clippingGroup.rasterize(150, false);
                let dataURL = raster.toDataURL();
                raster.remove();
                if (this.openResult) {
                    let image = new Image();
                    image.src = dataURL;
                    let w = window.open("");
                    w.document.write(image.outerHTML);
                    w.document.close();
                } else {
                    // trick to load image as a blob to be sent in FormData
                    fetch(dataURL).then(resolve => {
                        resolve.blob().then(blob => {
                            this.lastExport = blob;
                            this.$emit("export", this.lastExport);
                        });
                    });
                }
            }
        }
    };
</script>
<style lang="scss" scoped>
    .color-button {
        border-radius: 50%;
        width: 20px;
        height: 20px;
        cursor: pointer;
        display: inline-flex;
        transition: color 300ms;
    }

    .color-button.active:before {
        content: "\2713 ";
        color: white;
        font-weight: bold;
        position: absolute;
        left: 5px;
        bottom: 4px;
    }

    .size-button {
        cursor: pointer;
    }

    .button.size-button:focus {
        box-shadow: none;
    }

    .size-button:hover {
        filter: opacity(75%);
    }

    .size-button.active {
        background-color: #df691a;
        color: white;
    }

    .display-key {
        display: inline-block;
        margin-bottom: -15px;
        position: absolute;
        top: -2px;
        left: 30px;
    }

    .display-key.active {
        font-weight: bold;
    }

    .container-align-middle {
        display: flex;
        justify-content: center;
    }

    .set-cursor {
        cursor: pointer;
    }
</style>
