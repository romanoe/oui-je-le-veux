<script>
    import {onMount} from "svelte";
    import {geoPath, geoTransform} from "d3-geo";
    import {select, selectAll} from "d3-selection";
    import {draw, fade} from "svelte/transition";
    import {zoom, zoomIdentity} from "d3-zoom";
    import bbox from "@turf/bbox";
    import {scaleLinear, scaleSequential} from "d3-scale";
    import {extent, max} from "d3-array";
    import {interpolate} from "d3-interpolate";
    import legend from "d3-color-legend";
    import {Grid, Column, Row} from "carbon-components-svelte";


    let geodata = [], projection, path, colorScaleMariage, colorScaleConversions;
    let selected = 'total';

    function onChange(event) {
        selected = event.currentTarget.value;
    }


    // SVG properties
    const margin = {top: 50, right: 50, bottom: 0, left: 50},
        w = window.innerWidth / 1.5 - margin.left - margin.right,
        h = window.innerHeight / 1.5 - margin.top - margin.bottom;


    const doZoom = (e) => {
        selectAll("path").attr("transform", e.transform);
        selectAll("text").attr("transform", e.transform);
    }


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

    // Zoom
    const mapZoom = zoom().scaleExtent([1, 5]).on('zoom', doZoom);


    const initZoom = () => {
        select('svg')
            .call(mapZoom);
    }


    const zoomTo = (coordinates) => {
        select("svg").transition().duration(4000).call(mapZoom.transform, zoomIdentity.translate(w / 2, h / 2).scale(5).translate(-projection(coordinates)[0], -projection(coordinates)[1]));
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


    // Fetch data and initialize zoom
    onMount(async () => {
            const data = await fetch('./data/mariages_meme_sexe_2022_mn95.geojson').then((response) => response.json())

            // Thanks SRF data
            // https://blog.az.sg/posts/mapping-switzerland-2/
            // https://bl.ocks.org/mbostock/6216797

            const [minX, minY, maxX, maxY] = bbox(data);

            const width = w - margin.left - margin.right;
            // calculate aspect ratio and derive height
            const height = ((maxY - minY) / (maxX - minX)) * width;

            // Scales
            const x = scaleLinear()
                .range([0, width / 1.2])
                .domain([minX, maxX]);

            const y = scaleLinear()
                .range([0, height / 1.2])
                .domain([maxY, minY]);

            const maxMariages = max(data.features.map(d => d.properties.total / d.properties.pop * 1000))
            const maxConversions = max(data.features.map(d => d.properties.total_c / d.properties.pop * 1000))

            console.log(data.features.map(d => d))
            colorScaleMariage = scaleSequential(interpolate("white", "#50c1a3")).domain([0, maxMariages])
            colorScaleConversions = scaleSequential(interpolate("white", "#50c1a3")).domain([0, maxConversions])


            const projection = geoTransform({
                point: function (px, py) {
                    this.stream.point(x(px), y(py));
                },
            });


            path = geoPath().projection(projection);

            geodata = data.features


            initZoom();

        }
    )


</script>

<main>


    <Grid>

        <Row>

            <Column>

                <div id="map">
                    <svg width={w} height={h}>

                        <!--Cantons -->
                        {#key selected}
                            <g transition:fade>
                                {#each geodata as border, i}
                                    <g transform="translate({margin.left + margin.right},{margin.top + margin.bottom})">
                                        <path d={path(border)}
                                              fill="{selected == 'total' ? colorScaleMariage(border.properties[selected]/border.properties.pop*1000) : colorScaleConversions(border.properties[selected]/border.properties.pop*1000)}"
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
                    <p>Le dimanche 26 septembre 2021, la Suisse a dit <strong>oui</strong> au mariage civil pour toutes et tous.
                        Depuis juillet 2022, dans quel canton y a-t-il eu le plus de mariages du même sexe ? Et de conversions du partenariat enregistré ?

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


            </Column>
        </Row>
    </Grid>

    <footer><small><code>Données: <a
            href="https://www.bfs.admin.ch/bfs/fr/home/statistiques/population/mariages-partenaires-divorces/mariages.gnpdetail.2023-0216.html">Office
        fédéral de la statistique (OFS), Statistique du mouvement naturel de la
        population </a></code></small></footer>
</main>

<style>
    #title {
        padding-top: 10vh;
    }
    strong { font-weight: 1000; color: #50c1a3 }
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


</style>


