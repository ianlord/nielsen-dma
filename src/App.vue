<template>
    <main>
        <div class='tooltip' v-html='tooltipHtml' />
    </main>
</template>

<script>
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

// import nielsenTopoData from '@/data/nielsentopo.json'
// import tvData from '@/data/tv.json'

export default {
    data: () => ({
        tooltipHtml: ''
    }),
    mounted() {
        this.setupMap()

        window.addEventListener('resize', () => {
            document.querySelector('#map')?.remove()
            this.setupMap()
        })
    },
    methods: {
        setupMap() {
            const $vm = this
            const width = window.innerWidth
            const height = window.innerHeight

            // sets the type of view
            const projection = d3.geoAlbersUsa()
                .scale(width * 1.05) // size, bigger is bigger
                .translate([width / 2, height / 2])

            // creates a new geographic path generator
            const path = d3.geoPath().projection(projection)
            const xScale = d3.scaleLinear()
                .domain([0, 7])
                .range([0, 500])

            const xAxis = d3.axisBottom(xScale)
                .tickSize(13)
                .tickFormat(d3.format("0.0f"))


            // set svg window
            const svg = d3.select("body")
                .append("svg")
                .attr("width", width)
                .attr("height", height)
                .attr("id", "map")

            const graticule = d3.geoGraticule()
                .extent([[-98 - 45, 38 - 45], [-98 + 45, 38 + 45]])
                .step([5, 5])

            // adding a blank background
            const bg = svg.append("rect")
                .datum(graticule)
                .attr("width", width)
                .attr("height", height)
                .attr("class", "background")
                // .on("click", clicked);

            // declare g as our appended svg
            const g = svg.append("g")

            const defaultFill = "#aaa"

            d3.json('./data/nielsentopo.json', (error, dma) => {

                var nielsen = dma.objects.nielsen_dma.geometries;

                // adding data from tv json (number of TVs, etc) to map json
                d3.json('./data/tv.json', (error, tv) => {
                    for (var i = 0; i < nielsen.length; i++){
                        var dma_code = nielsen[i].id;
                        for (const key in tv[dma_code]) {
                            nielsen[i].properties[key] = tv[dma_code][key];
                        }
                    }

                    dma.objects.nielsen_dma.geometries = nielsen;

                    const states = g.append("g")
                        .attr("id", "dmas")
                        .selectAll("path")
                        .data(topojson.feature(dma, dma.objects.nielsen_dma).features)
                        .enter()
                        .append("path")
                        .attr("d", path)
                        .each(function (d) {
                            d3.select(this).attr("fill", d3.interpolateWarm($vm.mapRange(d.properties["TV Homes"], 100000, 2000000, 0, 1)))
                            // d3.select(this).attr("fill", d3.interpolateWarm($vm.mapRange(d.properties["Rank"], 210, 1, 0, 1)))
                        })
                        .on("mouseover", function (d) {
                            const prop = d.properties

                            d3.select(this).attr("class", "active")

                            console.log(prop)

                            const string = `
                                <p><strong>Metro Area Code:</strong> ${prop.dma}</p>
                                <p><strong>Metro Area Name:</strong> ${prop.dma1}</p>
                                <p><strong>Houses w/ TV:   </strong> ${$vm.numberWithCommas(prop["TV Homes"])}</p>
                                <p><strong>Houses w/ Cable:</strong> ${prop.cableperc}%</p>
                                <p><strong>Nielsen Rank:   </strong> #${prop.Rank}</p>`

                            $vm.tooltipHtml = string
                        })
                        .on("mouseout", function(d){
                            d3.select(this).attr("class", "")
                        })

                    // add dma borders
                    g.append("path", ".graticule")
                        .datum(topojson.mesh(dma, dma.objects.nielsen_dma, function(a, b) { 
                            return true;
                        }))
                        .attr("class", "dma-borders")
                        .attr("d", path)
                })
            })
        },

        numberWithCommas (x) {
            return x?.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",") || x || ''
        },

        clamp (input, min, max) {
            return input <= min ? min : input >= max ? max : input
        },

        mapRange (input, inMin, inMax, outMin, outMax, isClamp = true) {
            let val = ((input - inMin) * (outMax - outMin)) / (inMax - inMin) + outMin
            if (isClamp) val = this.clamp(val, outMin, outMax)
            return val
        }
    }
}
</script>