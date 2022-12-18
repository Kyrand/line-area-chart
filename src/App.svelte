<script>
	import { LayerCake, Svg, flatten } from 'layercake'
	import { stack, max, extent } from 'd3'
	import { scaleOrdinal } from 'd3-scale'
	import { format, precisionFixed } from 'd3-format'
	import { timeParse, timeFormat } from 'd3-time-format'
	import { schemeTableau10, rollup, group } from 'd3'

	import AxisX from './components/layercake/AxisX.svelte'
	import AxisY from './components/layercake/AxisY.svelte'
	import AreaStacked from './components/layercake/AreaStacked.svelte'

	import Switch from './components/Switch.svelte'
	import Range from './components/Range.svelte'
	import Test from './components/layercake/TestLC.svelte'
	import Legend from './components/Legend.svelte'

	// This example loads csv data as json using @rollup/plugin-dsv
	import fdata from './data/fruit.csv'
	import data from './data/unemployment.csv'

	let x = 'date'
	let y = 'industry'
	let z = 'unemployed'

	console.log('fdata: ', fdata)
	let _data = group(data, (d) => d[x])

	let sdata = []
	_data.forEach((g, key) => {
		let d = g.reduce(
			(cum, curr) => {
				cum[curr[y]] = curr[z]
				return cum
			},
			{ [x]: key }
		)
		sdata.push(d)
	})
	console.log('data: ', sdata)

	const xKey = 'date'
	const yKey = [0, 1]
	const zKey = 'key'

	const parseDate = timeParse('%Y-%m-%d')

	const seriesNames = Object.keys(sdata[0]).filter((d) => d !== xKey)
	//const seriesColors = ['#ff00cc', '#ff7ac7', '#ffb3c0', '#ffe4b8']
	const seriesColors = schemeTableau10

	let stackLines = false
	let normalizeAxes = false
	let t = 0
	let duration = 2000

	sdata.forEach((d) => {
		d[xKey] = typeof d[xKey] === 'string' ? parseDate(d[xKey]) : d[xKey]
		seriesNames.forEach((name) => {
			d[name] = +d[name]
		})
	})

	/* --------------------------------------------
	 * Create a stacked data structure
	 */
	const stackData = stack().keys(seriesNames)

	const _series = stackData(sdata)
	//let series = [..._series]
	let series = interpolateSeries(_series)

	const formatTickX = timeFormat('%b. %-d')
	const formatTickY = (d) => format(`.${precisionFixed(d)}s`)(d)

	let tweenedT = 1

	function processPt(pt, t) {
		let _y0 = pt[0]
		let _y1 = pt[1]
		let target1 = _y1 - _y0
		let target0 = target1 - 1
		let y0 = t * _y0 + (1 - t) * target0
		let y1 = t * _y1 + (1 - t) * target1
		let newPt = [y0, y1]
		newPt.data = pt.data
		return newPt
	}

	function interpolateSeries(_series, t = 0) {
		let series = _series.map((area) => {
			let pts = area.map((pt) => processPt(pt, t))
			pts.key = area.key
			return pts
		})
		return series
	}

	$: series = interpolateSeries(_series, tweenedT)
	$: t = stackLines ? 1 : 0
	$: _maxY = extent(flatten(flatten(_series)))[1]
	$: maxY = normalizeAxes ? null : _maxY

	$: console.log('Stack the lines: ', stackLines)
	$: console.log('Series: ', series)
	$: console.log('seriesNames: ', seriesNames)
	$: console.log('tweenedT: ', tweenedT)
	$: console.log('duration: ', duration)
</script>

<div class="_container">
	<h2>Transitioning from Stacked Area Chart to Line chart</h2>
	<h3>A Svelte, LayerCake, D3 production</h3>
	<p>
		This demo uses Svelte (+D3) tweening to transition from stacked areas to lines. You can switch
		to a normalized y axis and play with the duration (ms) of the transition. <br />
		The dataset used comes from Observable's
		<a href="https://observablehq.com/@d3/stacked-area-chart">Stacked Area chart demo</a> and shows US
		unemployed persons by industry 2000-2010.
	</p>
	<div class="app__wrapper">
		<div class="ui">
			<Switch bind:flag={stackLines} label="Stack the lines" id="stackLines" />
			<Switch bind:flag={normalizeAxes} label="Dynamic y axis" id="normAxes" />
			<Range bind:value={duration} min={0} max={5000} label="Transition duration (ms)" />
		</div>
		<div class="chart-container">
			<div class="legend__wrapper">
				<Legend labels={seriesNames} colorScale={seriesColors} />
			</div>

			<LayerCake
				padding={{ top: 0, right: 0, bottom: 20, left: 17 }}
				x={(d) => d.data[xKey]}
				y={yKey}
				z={zKey}
				yDomain={[0, maxY]}
				zScale={scaleOrdinal()}
				zDomain={seriesNames}
				zRange={seriesColors}
				flatData={flatten(series)}
				data={series}
			>
				<Svg>
					<AxisX formatTick={formatTickX} tickMarks={true} />
					<AxisY _formatTick={formatTickY} ticksEvery="100" label="Unemployed persons" />
					<AreaStacked />
					<Test {t} bind:tweenedT {duration} />
				</Svg>
			</LayerCake>
		</div>
	</div>
</div>

<style>
	/*
    The wrapper div needs to have an explicit width and height in CSS.
    It can also be a flexbox child or CSS grid element.
    The point being it needs dimensions since the <LayerCake> element will
    expand to fill it.
  */
	.app__wrapper {
		display: flex;
		flex-direction: row;
		gap: 4em;
	}
	.chart-container {
		width: 100%;
		height: 600px;
	}
	.ui {
		min-width: 220px;
	}
</style>
