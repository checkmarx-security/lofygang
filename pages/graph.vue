<template>
    <div>
        <div ref="container" class="container">
        </div>
        <div class="no-data" v-if="!filters.IOCs.length || !filters.keywords.length || filteredItems.length === 0">
            <div class="no-data__title">No Results</div>
            <div class="no-data__subtitle">
                <div class="no-data__button" @click="resetFilters">Click here</div>
                to reset filters
            </div>
        </div>
    </div>
</template>

<script>
import * as d3 from 'd3';
import * as PIXI from 'pixi.js';
import {Viewport} from 'pixi-viewport';
import {Cull} from '@pixi-essentials/cull';
import lofyIcon from '../static/lofylogo.svg?raw'
import activePackageIconPNG from '../static/activePackage.png'
import packageIconPNG from '../static/package.png'
import serverIconPNG from '../static/serverIcon.png'
import userIconPNG from '../static/userIcon.png'
import githubIconPNG from '../static/githubIcon.png'
import forceInABox from 'force-in-a-box';


export default {
    name: "graph",
    components: {},
    props: ['items', 'filters'],
    data() {
        return {
            pixiObjects: {},
            noData: false,
            graphWidth: 900,
            windowWidth: 900,
            texturesDict: {
                'package': activePackageIconPNG,
                'email': githubIconPNG,
                'username': userIconPNG,
                'IOCs': serverIconPNG,
                'lofygang': lofyIcon,
                'group': lofyIcon
            }
        }
    },
    computed: {
        filteredItems() {
            let IOCsFilter = this.filters.IOCs;
            let IOCsMap = this.filters.IOCsMap;
            let keywordsFilter = this.filters.keywords;
            let items = this.items;
            if (Object.values(IOCsMap).length || keywordsFilter.length) {
                items = items.filter((item) => {
                    let keywords = item.group
                    let IOCs = item.servers
                    if (!keywords.some((word) => keywordsFilter.includes(word))) {
                        return false
                    }
                    if (!IOCsFilter.some((ioc) => IOCsMap[ioc].includes(item))) {
                        return false
                    }
                    return true
                })
            }
            return items
        },
        graphData() {
            let items = Object.values(this.filteredItems);
            let links = []
            let nodesMap = {}
            for (let item of items) {
                let node_id = `${item.name}/${item.version}`
                nodesMap[node_id] = item
                nodesMap[node_id].icon = "package"
                nodesMap[item.username] = {
                    id: `${item.username}/${item.version}`,
                    name: item.username,
                    type: item.type,
                    group: item.group,
                    version: item.version,
                    email: item.email,
                    username: item.username,
                    date: item.date,
                    IOCs: item.servers,
                    icon: "username"
                }
                links.push({source: {id: node_id}, target: {id: `${item.username}/${item.version}`}})
                item.email.forEach((mail) => {
                    nodesMap[mail] = {
                        name: mail,
                        type: item.type,
                        group: item.group,
                        version: item.version,
                        email: mail,
                        username: item.username,
                        date: item.date,
                        IOCs: item.servers,
                        icon: "email"
                    }
                    links.push({source: {id: node_id}, target: {id: `${item.mail}/${item.version}`}})
                })
                item.servers.forEach((ioc) => {
                    nodesMap[ioc] = {
                        name: ioc,
                        type: item.type,
                        group: item.group,
                        version: item.version,
                        email: item.email,
                        username: item.username,
                        date: item.date,
                        IOCs: ioc,
                        icon: "IOCs"
                    }
                    links.push({source: {id: node_id}, target: {id: `${ioc}/${item.version}`}})
                })
                for (let group of item.group) {
                    if (group === "none") {
                        continue
                    }
                    nodesMap[group] = {
                        name: group,
                        type: item.type,
                        group: group,
                        version: item.version,
                        email: item.email,
                        username: item.username,
                        date: item.date,
                        IOCs: item.servers,
                        icon: "group"
                    }
                    links.push({source: {id: node_id}, target: {id: `${group}/${item.version}`}})
                }
            }
            return {nodes: nodesMap, links: links}
        },
        useViewport() {
            return this.viewport
        },
        useSimulation() {
            return this.simulation
        },
    },
    mounted() {
        this.initGraph()
        let vm = this
        window.addEventListener('resize', function () {
            vm.windowWidth = window.innerWidth
            vm.graphWidth = vm.$parent.$refs.packages.clientWidth - 230
        });

    },
    methods: {
        initGraph() {

            if (!this.filters.IOCs || !this.filters.keywords) {
                this.noData = true
                return
            }
            let {nodes, links} = this.graphData
            let container = this.$refs.container;


            let height;
            let width;
            let containerRect;

            containerRect = container.getBoundingClientRect()
            height = containerRect.height;
            width = this.$parent.$refs.packages.clientWidth ? this.$parent.$refs.packages.clientWidth - 230 : this.graphWidth;

            let dragged = false;
            container.innerHTML = "";

            let app = new PIXI.Application({
                width,
                height,
                antialias: !0,
                transparent: !0,
                resolution: 1,
                autoResize: true
            });
            container.appendChild(app.view)
            container.firstElementChild.setAttribute("ref", "canvas")

            let viewport = new Viewport({
                screenWidth: width,
                screenHeight: height,
                worldWidth: width * 4,
                worldHeight: height * 4,
                passiveWheel: false,

                interaction: app.renderer.plugins.interaction
            });
            this.viewport = viewport

            app.stage.addChild(viewport);
            viewport.drag().pinch().wheel().decelerate().clampZoom({minWidth: width / 4, minHeight: height / 4});
            viewport.fit(true, this.graphWidth * 3, 900 * 3)

            let groupingForce = forceInABox()
                .strength(0.1)
                .groupBy("group")
                .enableGrouping(true)
                .template('force')
                .links(this.graphData.links)
                .forceNodeSize(d => 60)
                .size([width, height])

            const simulation = d3.forceSimulation()
                .nodes(Object.values(this.graphData.nodes))
                .force('link', d3.forceLink(this.graphData.links).id((d) => {
                    return d.name
                }))
                .force('charge', d3.forceManyBody().strength(-100))
                .force("group", groupingForce)
                .force('center', d3.forceCenter(width / 2, height / 2))
                .force("collide", d3.forceCollide().strength(1).radius(60).iterations(30))

            this.simulation = simulation

            const cull = new Cull().addAll(viewport.children);
            let cullDirty = false;
            viewport.on('frame-end', function () {
                if (viewport.dirty || cullDirty) {
                    cull.cull(app.renderer.screen);
                    viewport.dirty = false;
                    cullDirty = false;
                }
            })

            viewport.children.forEach((child) => child.destroy(true))

            let visualLinks = new PIXI.Graphics();
            viewport.addChild(visualLinks);
            let currDraggedNode;
            let vm = this;
            this.dragStart = function onDragStart(evt) {
                this.currPoint = {x: evt.data.originalEvent.x, y: evt.data.originalEvent.y}
                viewport.plugins.pause('drag');
                simulation.alphaTarget(0.3).restart();
                this.isDown = true;
                this.eventData = evt.data;
                this.alpha = 1;
                this.dragging = true;
            }

            this.dragEnd = function onDragEnd(evt) {
                if (this.currPoint.x === evt.data.originalEvent.x && this.currPoint.y === evt.data.originalEvent.y) {
                    if (vm.graphData.nodes[vm.currDraggedNode.name].icon !== "server" && vm.graphData.nodes[vm.currDraggedNode.name].icon !== "user") {
                        vm.$emit('itemSelected', vm.graphData.nodes[vm.currDraggedNode.name])
                    }
                }
                evt.stopPropagation();
                if (!evt.active) simulation.alphaTarget(0.1);
                this.alpha = 1;
                this.dragging = false;
                this.isOver = false;
                this.eventData = null;
                viewport.plugins.resume('drag');
            }

            this.dragMove = function onDragMove(node, gfx) {
                if (gfx.dragging) {
                    dragged = true;
                    const newPosition = gfx.eventData.getLocalPosition(gfx.parent);
                    this.x = newPosition.x;
                    this.y = newPosition.y;
                }
            }

            this.linksGraphics = {}
            const ticked = () => {
                Object.values(this.pixiObjects).forEach((node) => {
                    let x = node.x
                    let y = node.y
                    node.gfx.position = new PIXI.Point(x, y);
                });

                for (let i = visualLinks.children.length - 1; i >= 0; i--) {
                    visualLinks.children[i].destroy();
                }

                if (width !== this.graphWidth) {
                    this.viewport.moveCenter(this.graphWidth / 2, 444 / 2);
                    width = this.graphWidth
                }

                visualLinks.clear();
                visualLinks.removeChildren();
                visualLinks.alpha = 0;
                this.graphData.links.forEach((link) => {
                    let source = link.source.id;
                    let target = link.target.id;
                    if (!this.pixiObjects[source] || !this.pixiObjects[target]) {
                        return
                    }
                    let sourceNode = this.pixiObjects[source]
                    let targetNode = this.pixiObjects[target]
                    if (sourceNode.gfx.visible && targetNode.gfx.visible && viewport.lastViewport.scaleY > 0.1) {
                        visualLinks.alpha = 1
                    }
                    visualLinks.lineStyle(2, 0xaaaaaa);
                    visualLinks.moveTo(sourceNode.x, sourceNode.y);
                    visualLinks.lineTo(targetNode.x, targetNode.y);
                    visualLinks.name = `${sourceNode.id}###${targetNode.id}`
                    this.linksGraphics[visualLinks.name] = visualLinks
                });
                visualLinks.endFill();
            }
            this.viewport = viewport
            this.app = app
            this.simulation.on("tick", ticked);
            this.simulation.tick();
        },
        calculateAspectRatioFit(srcWidth, srcHeight, maxWidth, maxHeight) {
            console.log(srcWidth, srcHeight, maxWidth, maxHeight)
            var ratio = Math.min(maxWidth / srcWidth, maxHeight / srcHeight);
            return {width: srcWidth * ratio, height: srcHeight * ratio};
        },
        restartForce() {
            let sim = this.useSimulation
            let nodes = this.graphData.nodes
            let links = this.graphData.links
            sim.nodes(Object.values(nodes)).restart()
            sim.force("link").links(links)
            sim.alpha(1).restart()
        },
        resetFilters() {
            this.$emit('resetFilters')
        }
    },
    watch: {
        graphData() {
            console.log(this.filters.IOCs)
            console.log(this.filters.keywords)
            if (!this.filters.IOCs || !this.filters.keywords) {
                this.noData = true
                return
            }
            let vm = this;
            let dragged = false;
            let viewport = this.useViewport
            viewport.children.length = 1;
            let nodes = this.graphData.nodes
            Object.values(nodes).forEach((node) => {
                const boundDrag = this.dragMove.bind(node);
                let item = node;
                if (node.icon === 'group') {
                    item.gfx = new PIXI.Text(node.group);
                    item.gfx.style = new PIXI.TextStyle({fontFamily: 'Roboto', fontSize: 40, fill: 0x000000, align: 'center'})
                } else if (node.icon === "package") {
                    let icon = node.available ? activePackageIconPNG : packageIconPNG
                    item.gfx = new PIXI.Sprite.from(icon);
                } else {
                    item.gfx = new PIXI.Sprite.from(this.texturesDict[node.icon]);
                }
                let nodeID = `${node.name}/${node.version}`
                item.gfx.id = nodeID;
                let currName = node.name
                item.name = currName
                item.gfx.anchor.set(0.5)
                if (node.icon !== 'group') {
                    let icon_ratio = 1.1679
                    if (node.icon === 'package') {
                        icon_ratio = node.available ? 157 / 178 : 229 / 254
                        item.gfx.width = node.available ? 60 : 80;
                        item.gfx.height = (node.available ? 60 : 80)  / icon_ratio;
                    } else if (node.icon === "IOCs") {
                        icon_ratio =  148 / 189
                        item.gfx.width = 60;
                        item.gfx.height = 60 * icon_ratio;
                    } else if (node.icon === "email") {
                        icon_ratio = 1
                        item.gfx.width = 60;
                        item.gfx.height = 60 * icon_ratio;
                    } else if (node.icon === "username") {
                        icon_ratio = 132 / 142
                        item.gfx.width = 60;
                        item.gfx.height = 60 * icon_ratio;
                    }

                    item.gfx.on('mouseover', (mouseData) => {
                        vm.currDraggedNode = node
                        const text = new PIXI.Text(currName, {
                            fontSize: 24,
                            fontFamily: "roboto",
                            fontWeight: 700,
                            fill: '#000',
                            strokeThickness: 4,
                            stroke: '#fff'
                        });
                        text.anchor.set(0.5);
                        text.resolution = 2;
                        text.position.y = -100;
                        item.gfx.addChild(text);
                    });
                    item.gfx.on('mouseout', () => {
                        node.gfx.removeChildren()
                    });
                }
                item.gfx
                    .on('click', (e) => {
                        if (!dragged) {
                            e.stopPropagation();
                        }
                        dragged = false;
                    })
                    .on('mousedown', this.dragStart)
                    .on('mouseup', this.dragEnd)
                    .on('mouseupoutside', this.dragEnd)
                    .on('mousemove', () => boundDrag(node, node.gfx));
                item.gfx.type = node.icon
                viewport.addChild(node.gfx);

                item.gfx.interactive = true;
                item.gfx.buttonMode = true;
                item.gfx.hitArea = new PIXI.Rectangle(-40, -40, 60, 60);
                this.pixiObjects[nodeID] = item
            });
            viewport.fit(true, this.graphWidth * 3, 900 * 3)
            this.viewport = viewport
            let links = this.graphData.links
            this.useSimulation.alpha(1).restart();
            this.restartForce()
            this.noData = Object.keys(this.graphData.nodes).length === 0
        },
        graphWidth() {
            if (this.windowWidth < 1200) {
                this.app.renderer.resize(this.graphWidth, 400)
                this.viewport.moveCenter(this.graphWidth / 2, 400 / 2)
            } else {
                this.app.renderer.resize(this.graphWidth, 444)
                this.viewport.moveCenter(this.graphWidth / 2, 444 / 2)
            }
            this.viewport.fit(true, this.graphWidth * 3, 900 * 3)
            this.useSimulation.alpha(1).restart();
        }
    }

}
</script>

<style scoped lang="scss">
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;700&display=swap');

.container {
    width: 100%;
    height: 100%;
    overflow: hidden;
}

.no-data {
    position: absolute;
    font-family: "roboto", sans-serif;
    color: black;
    z-index: 10;
    font-size: 20px;
    display: flex;
    flex-direction: column;
    place-items: center;
    place-content: center;
    user-select: none;

    &__title {
        font-size: 24px;
        font-weight: bold;
        color: #B20000;
        padding-bottom: 4px;
    }

    &__subtitle {
        font-size: 16px;
        font-weight: lighter;
        color: #4C0000;
    }

    &__button {
        display: inline-block;
        text-decoration: underline;
        cursor: pointer;

        &:hover {
            opacity: 0.6;
        }
    }
}
</style>
