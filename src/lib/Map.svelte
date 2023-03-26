<script>
    import {onMount} from "svelte";
    import {geoPath, geoTransform} from "d3-geo";
    import {select, selectAll} from "d3-selection";
    import {fade} from "svelte/transition";
    import bbox from "@turf/bbox";
    import {scaleLinear, scaleSequential, scaleBand} from "d3-scale";
    import {max, sum} from "d3-array";
    import {interpolate} from "d3-interpolate";
    import {Grid, Column, Row} from "carbon-components-svelte";
    import {axisBottom} from "d3-axis";


    let geodata = [], dataAgg = null, data_total = [], projection, path, colorScaleMariage, colorScaleConversions,
        linearScale, nConversions, nMariages;
    let selected = 'total';

    function onChange(event) {
        selected = event.currentTarget.value;
    }


    // SVG properties
    const margin = {top: 50, right: 20, bottom: 0, left: 20},
        w = window.innerWidth * 0.6 - margin.left - margin.right,
        h = window.innerHeight * 0.9 - margin.top - margin.bottom;


    // Tooltip
    const showTooltip = (event, source_id) => {
        let tooltip = document.getElementById(source_id);
        tooltip.style.display = "block";
        tooltip.style.left = event.layerX + 10 + 'px';
        tooltip.style.top = event.layerY + 10 + 'px';
    }
    const hideTooltip = (event, source_id) => {
        let tooltip = document.getElementById(source_id);
        tooltip.style.display = "none";
    }

    // Mouse over : change color of path
    const colors = ["#fdf21f", "#3fdf4b", "#3462fe", "#fc0049", "#fe8f01"]
    let color = "#fdf21f";

    function handleMouseOver(e, canton_id) {
        selectAll("path").attr("stroke-width", 0);
        const selected_path = document.getElementById(canton_id)
        color = colors[Math.floor(Math.random() * colors.length)]
        selected_path.style.stroke = color;
        selected_path.style.strokeWidth = 1.5;
    }

    function handleMouseOut(e, canton_id) {
        const selected_path = document.getElementById(canton_id)
        selected_path.style.strokeWidth = 0;
        selected_path.style.stroke = "white";
    }


    const setColorScale = (data, property) => {

        const maxProperty = max(data.map(d => d.properties[property] / d.properties.pop * 10000))

        // Color scales and legend
        return scaleSequential(interpolate("white", "#50c1a3")).domain([0, maxProperty])

    }

    $: colorScale = setColorScale(geodata, selected)


    const setLegendAxis = (domain) => {

        // Axis
        select("#legend").append('g').attr("class", "axis").attr("transform", "translate(0,35)")

        // linear scale for axis
        const sequentialScale = scaleSequential()
            .domain(domain)
            .range([0, 300])
            .nice()

        // Construct axis
        const axis = axisBottom(sequentialScale).tickSize([-20]);
        select('.axis')
            .call(axis);
    }


    $: axisLegend = setLegendAxis(colorScale.domain())


    // Fetch and manipulate data
    onMount(async () => {
            const data = await fetch('./data/mariages_meme_sexe_2022_mn95.geojson').then((response) => response.json())
            geodata = data.features


            // Map MN95 data
            // Thanks SRF data + Luc Guillemot
            // https://blog.az.sg/posts/mapping-switzerland-2/
            // https://bl.ocks.org/mbostock/6216797
            const [minX, minY, maxX, maxY] = bbox(data);

            const width = w - margin.left - margin.right;
            // calculate aspect ratio and derive height
            const height = ((maxY - minY) / (maxX - minX)) * width;

            // Scales
            const x = scaleLinear()
                .range([0, width])
                .domain([minX, maxX]);

            const y = scaleLinear()
                .range([0, height])
                .domain([maxY, minY]);


            // Map projection
            const projection = geoTransform({
                point: function (px, py) {
                    this.stream.point(x(px), y(py));
                },
            });


            path = geoPath().projection(projection);


            // Barchart data

            const totalConversions = [
                {
                    sexe: "Femmes",
                    value: sum(geodata.map(d => d.properties.c_f_f))

                },
                {
                    sexe: "Hommes",
                    value: sum(geodata.map(d => d.properties.c_h_h))
                }
            ];

            nConversions = totalConversions.map(d => d.value).reduce((acc, currentValue) => acc + currentValue, 0);


            const totalMariages = [
                {
                    sexe: "Femmes",
                    value: sum(geodata.map(d => d.properties.f_f))

                },
                {
                    sexe: "Hommes",
                    value: sum(geodata.map(d => d.properties.h_h))
                }
            ];

            nMariages = totalMariages.map(d => d.value).reduce((acc, currentValue) => acc + currentValue, 0);


            dataAgg = [{
                total_c: totalConversions,
                total: totalMariages
            }]


            linearScale = scaleLinear().domain([0, sum(geodata.map(d => d.properties.c_h_h))]).range([heightB - padding.bottom, padding.top])


        }
    )


    // Statistiques globales
    $: if (dataAgg != null) {
        data_total = dataAgg.map(d => d[selected])[0]
    }
    ;

    let widthB = 300;

    const padding = {top: 20, right: 15, bottom: 20, left: 25};

    let heightB = 200 + padding.bottom + padding.top;


    $: innerWidth = widthB - (padding.left + padding.right);
    $: bandScale = scaleBand().domain(data_total.map(d => d.sexe)).range([padding.left, widthB - padding.right]).paddingOuter(0.5).paddingInner(0.7);


</script>

<main>

    <Grid>

        <Row>

            <Column>

                <div id="map">
                    <svg width={w} height={h+margin.top+margin.bottom} id="svg-map">
                        <defs>
                            <linearGradient id="MyGradient">
                                <stop offset="0%" stop-color="white"/>
                                <stop offset="100%" stop-color="#50c1a3"/>
                            </linearGradient>
                        </defs>

                        <!--Cantons -->
                        {#key selected}
                            <g transition:fade>
                                {#each geodata as border, i}
                                    <g transform="translate({margin.left + margin.right},{margin.top + margin.bottom})">
                                        <path d={path(border)}
                                              fill="{colorScale(border.properties[selected]/border.properties.pop*10000)}"
                                              on:mouseover={showTooltip(event, border.properties.canton)}
                                              on:mouseout={hideTooltip(event, border.properties.canton)}
                                              on:blur={hideTooltip(event, border.properties.canton)}
                                              on:focus={showTooltip(event, border.properties.canton)}
                                              id={"path-"+border.properties.canton}
                                              on:mouseenter={handleMouseOver(event,"path-"+border.properties.canton)}
                                              on:mouseleave={handleMouseOut(event,"path-"+border.properties.canton)}
                                        ></path>
                                    </g>
                                {/each}

                            </g>
                        {/key}

                        <g id="legend" transform="translate({w/3},{ h - margin.top - margin.left})">

                            {#if selected === "total"}
                                <text>Nombre de mariages pour 10'000 habitants</text>
                            {:else}
                                <text>Nombre de conversions pour 10'000 habitants</text>
                            {/if}

                            <g transform="translate(0,15)">
                                <rect width="300" height="20" fill="url(#MyGradient)"></rect>
                            </g>


                        </g>


                    </svg>


                    {#each geodata as kanton, i}
                        <div id={kanton.properties.canton}
                             style="position: absolute; display: none; background-color: whitesmoke; opacity: 0.6; padding: 20px; border-radius: 10%;">
                            <h5 style="color:{color}">{kanton.properties.canton}</h5>
                            Femmes: <b>{kanton.properties.f_f}, conversion {kanton.properties.c_f_f} </b><br>
                            Hommes: <b>{kanton.properties.h_h}, conversion {kanton.properties.c_h_h} </b><br>
                        </div>

                    {/each}
                </div>


            </Column>

            <Column>
                <div id="title">
                    <h2>Oui, je le veux </h2>
                    <br>
                    <p>Le dimanche 26 septembre 2021, la Suisse a dit <strong>oui</strong> au mariage civil pour toutes
                        et tous.
                        Depuis juillet 2022, {nMariages} mariages de même sexe et {nConversions} conversions de
                        partenariat enregistré ont eu lieu, mais dans quel canton y en a-t-il eu le plus ?

                        <strong>Découvrons ensemble</strong> !</p>
                </div>
                <br>
                <br>
                <div id="labels">
                    <label>
                        <input checked={selected==='total'} on:change={onChange} type="radio" name="mariage"
                               value="total"/> Mariages
                    </label>

                    <label>
                        <input checked={selected==='total_c'} on:change={onChange} type="radio" name="mariage"
                               value="total_c"/> Conversion partenariat enregistré
                    </label>
                </div>


                <svg width="{widthB}" height="{heightB+padding.bottom}" id="barchart">
                    {#key selected}
                        {#each data_total as feature, i}
                            <g>
                                <rect
                                        x="{bandScale(feature.sexe)}"
                                        y="{linearScale(feature.value)}"
                                        width="{bandScale.bandwidth()}"
                                        height="{heightB - padding.bottom - linearScale(feature.value)}"
                                        fill="#50c1a3"
                                ></rect>

                                <text class="axisLabel" x="{bandScale(feature.sexe) + bandScale.bandwidth()/2}"
                                      y="{linearScale(feature.value-150)}" fill="white">{feature.value}</text>
                            </g>

                            <text class="axisLabel" x="{bandScale(feature.sexe) + bandScale.bandwidth()/2}"
                                  y="{heightB}" fill="#C7287D" back>{feature.sexe}</text>

                        {/each}
                    {/key}
                </svg>
            </Column>
        </Row>
    </Grid>


    <footer><small><code>Données: <a
            href="https://www.bfs.admin.ch/bfs/fr/home/statistiques/population/mariages-partenaires-divorces/mariages.gnpdetail.2023-0216.html">Office
        fédéral de la statistique (OFS), Statistique du mouvement naturel de la
        population </a></code></small></footer>
</main>

<style>


    #legend {
        font-family: "monospace";
        font-weight: 10;
        color: #C7287D;
        fill: #C7287D;
    }

    #title {
        padding-top: 10vh;
    }

    strong {
        font-weight: 1000;
        color: #50c1a3
    }

    footer {
        text-align: right;
        right: 0;
        bottom: 0;
        margin: 20px;
        position: fixed;
    }

    p {
        color: #C7287D;
    }

    label {
        display: block;
        font-size: 1vw;
        line-height: 2vw;
    }

    #labels {
        color: #C7287D;


    }

    a {
        color: #C7287D;
        text-decoration: none;
    }


    h2 {
        color: #C7287D;
        font-family: "Roboto", sans-serif;
        text-transform: uppercase;
        font-size: 3.3vw;
        letter-spacing: 0.2em;
        cursor: pointer;
        text-shadow: 0.04em 0.04em #fc0049, 0.08em 0.08em #fe8f01,
        0.12em 0.12em #fdf21f, 0.16em 0.16em #3fdf4b, 0.2em 0.2em #3462fe;
    }

    .axisLabel {
        text-anchor: middle;
    }

</style>


