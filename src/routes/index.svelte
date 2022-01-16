<script type="text/typescript">
	import { Delaunay } from 'd3-delaunay';
	import { onMount } from 'svelte';
	import {hsv} from 'd3-hsv';
	import * as d3 from 'd3-color';

	// the animation will aim to run at this framerate
	const TARGET_FPS = 30;

	// if the animation slows below CUTOFF_FPS for more than
	// CUTOFF_MIN_FRAMES frames, the animation will stop
	const CUTOFF_FPS = 25;
	const CUTOFF_MIN_FRAMES = 10;

	// rovers will appear and disappear this many pixels past the edge of the canvas
	const CLIP_DISTANCE = 100;

	// after MAX_ROVERS is exceeded, the first rover will be removed
	const MAX_ROVERS = 10;

	class GameBoard {
		fixedPoints: Point[];
		rovers: Rover[];
		context: CanvasRenderingContext2D;
		roverIntervalSeconds: number;
		previousFrameTimestampMs: number;
		cutoffFramerate: number;
		cutoffMinFrames: number;
		numFramesBelowCutoff = 0;
		stopped = false;

		constructor(
			context: CanvasRenderingContext2D,
			numFixedPoints: number = 10,
			roverIntervalSeconds: number = 2,
			cutoffFramerate: number = CUTOFF_FPS,
			cutoffMinFrames: number = CUTOFF_MIN_FRAMES
		) {
			this.context = context;
			this.rovers = [];
			this.cutoffFramerate = cutoffFramerate;
			this.cutoffMinFrames = cutoffMinFrames;
			this.roverIntervalSeconds = roverIntervalSeconds;
			this.previousFrameTimestampMs = Date.now();

			// initialize stationary points
			this.fixedPoints = [];
			for (let i = 0; i < numFixedPoints; i++) {
				const x = Math.random() * context.canvas.clientWidth;
				const y = Math.random() * context.canvas.clientHeight;
				const color = RGB.randomDark();
				this.fixedPoints.push(new Point(new Vector2(x, y), color));
			}
		}

		loop() {
			// board bounds
			const xmax = this.context.canvas.clientWidth;
			const ymax = this.context.canvas.clientHeight;
			// calculate fps
			const currentFrameTimestampMs = Date.now();
			const timeElapsedMs = currentFrameTimestampMs - this.previousFrameTimestampMs;
			this.previousFrameTimestampMs = currentFrameTimestampMs;
			const fps = 1000 / timeElapsedMs;
			// warn if fps is below cutoff threshold
			if (fps < this.cutoffFramerate) {
				console.warn(`fps is ${fps}, below threshold of ${this.cutoffFramerate}`);
			}
			// if fps is too low for too many frames, stop the animation
			if (fps > this.cutoffFramerate) {
				this.numFramesBelowCutoff = 0;
			} else {
				this.numFramesBelowCutoff++;
			}
			if (this.numFramesBelowCutoff > this.cutoffMinFrames) {
				this.stopped = true;
			}
			if (this.stopped) {
				console.debug(
					`Ran under ${this.cutoffFramerate} fps for ${this.cutoffMinFrames} frames, stopping animation`
				);
				return;
			}

			// prune excess rovers
			if (this.rovers.length > MAX_ROVERS) {
				this.rovers.splice(0, 1);
				console.debug("Too many rovers, removing the oldest one");
			}

			// possibly spawn a new rover
			if (Math.random() < 1 / (TARGET_FPS * this.roverIntervalSeconds)) {
				// the rover is spawned by picking a location for it to
				// pass through, picking the angle at which it will pass
				// through the location, and then figuring out where to
				// spawn it
				const location = new Vector2(
					Math.random() * this.context.canvas.clientWidth,
					Math.random() * this.context.canvas.clientHeight
				);
				const angle = Math.random() * 2 * Math.PI;
				// slope and y-intercept of the vector create by angle passing through location
				const m = Math.sin(angle) / Math.cos(angle);
				const b = location.y - m * location.x;
				// possible spawn and despawn points along the vector are...
				const xUp = -b / m; // y = 0
				const xDown = (ymax - b) / m; // y = ymax
				const yLeft = b; // x = 0
				const yRight = m * xmax + b; // x = xmax
				const pointsOfInterest = [
					new Vector2(xUp, 0),
					new Vector2(xDown, ymax),
					new Vector2(0, yLeft),
					new Vector2(xmax, yRight)
				];
				// unless the line is perfectly vertical, sorting by x coord lets us pick the points at either edge
				pointsOfInterest.sort((a, b) => a.x - b.x);
				// randomly pick one extreme point to be the spawn and the other to be the despawn
				let spawn: Vector2, despawn: Vector2;
				if (Math.random() < 0.5) {
					spawn = pointsOfInterest[0];
					despawn = pointsOfInterest[pointsOfInterest.length - 1];
				} else {
					spawn = pointsOfInterest[pointsOfInterest.length - 1];
					despawn = pointsOfInterest[0];
				}
				// if the spawn is beyond the clip distance, move it inward
				spawn.x = Math.max(spawn.x, 0 - CLIP_DISTANCE);
				spawn.x = Math.min(spawn.x, xmax + CLIP_DISTANCE);
				spawn.y = Math.max(spawn.y, 0 - CLIP_DISTANCE);
				spawn.y = Math.min(spawn.y, ymax + CLIP_DISTANCE);
				// spawn the rover
				this.rovers.push(
					new Rover(
						new Vector2(spawn.x, spawn.y),
						spawn,
						despawn,
						Math.random() * 50 + 50,
						RGB.randomBright()
					)
				);
			}

			// move each rover
			this.rovers.forEach((rover) => {
				// total x and y distance the rover needs to travel
				const dTotal = rover.despawn.minus(rover.spawn);
				const angle = Math.atan(dTotal.y / dTotal.x);
				// distance to travel this frame
				const distanceThisFrame = rover.speed / TARGET_FPS;
				const d = new Vector2(
					distanceThisFrame * Math.cos(angle),
					distanceThisFrame * Math.sin(angle)
				);
				// apply location transform
				rover.location = rover.location.plus(d);
				// if the rover is beyond the clip distance, delete it
				if (
					rover.location.x < 0 - CLIP_DISTANCE ||
					rover.location.x > xmax + CLIP_DISTANCE ||
					rover.location.y < 0 - CLIP_DISTANCE ||
					rover.location.y > ymax + CLIP_DISTANCE
				) {
					this.rovers.splice(this.rovers.indexOf(rover), 1);
				}
			});

			// render rovers and cells
			// concat fixed point colors and rover colors
			const colors: d3.Color[] = this.fixedPoints
				.map((point) => point.color)
				.concat(this.rovers.map((rover) => rover.color));
			// concat fixed point and rover locations
			const locations: Vector2[] = this.fixedPoints
				.map((point) => point.location)
				.concat(this.rovers.map((rover) => rover.location));
			// get a Delaunay triangulation of our points
			const delaunay = Delaunay.from(
				locations,
				(point) => point.x,
				(point) => point.y
			);
			// get the voronoi diagram, with the bounds set to the edges of the canvas
			const voronoi = delaunay.voronoi([
				0 - CLIP_DISTANCE,
				0 - CLIP_DISTANCE,
				xmax + CLIP_DISTANCE,
				ymax + CLIP_DISTANCE
			]);
			// render each cell
			for (let i = 0; i < colors.length; i++) {
				this.context.fillStyle = colors[i].toString();
				const polygon = voronoi.cellPolygon(i);
				if (polygon) {
					this.context.beginPath();
					polygon.forEach((vertex: [x: number, y: number], index) => {
						if (index == 0) {
							this.context.moveTo(vertex[0], vertex[1]);
						} else {
							this.context.lineTo(vertex[0], vertex[1]);
						}
					});
					this.context.closePath();
					this.context.fill();
				}
			}

			// sleep
			setTimeout(() => {
				this.loop();
			}, 1000 / TARGET_FPS);
		}
	}

	class Point {
		location: Vector2;
		color: d3.Color;

		constructor(location: Vector2, color: d3.Color = null) {
			this.location = location;
			this.color = color ? color : RGB.random();
		}

		toString() {
			return this.location.toString();
		}
	}

	class Rover extends Point {
		spawn: Vector2;
		despawn: Vector2;
		speed: number;

		constructor(
			location: Vector2,
			spawn: Vector2,
			despawn: Vector2,
			speed: number = 100,
			color: d3.Color = null
		) {
			super(location, color);
			this.spawn = spawn;
			this.despawn = despawn;
			this.speed = speed;
		}
	}

	type Polygon = [x: number, y: number][];

	class Vector2 {
		x: number;
		y: number;

		constructor(x: number, y: number) {
			this.x = x;
			this.y = y;
		}

		toString() {
			return `(${this.x}, ${this.y})`;
		}

		minus(other: Vector2): Vector2 {
			return new Vector2(this.x - other.x, this.y - other.y);
		}

		plus(other: Vector2): Vector2 {
			return new Vector2(this.x + other.x, this.y + other.y);
		}
	}

	class RGB {
		r: number;
		g: number;
		b: number;

		/**
		 * Initialize a new RGB value
		 * @param r 0...1
		 * @param g 0...1
		 * @param b 0...1
		 */
		constructor(r: number, g: number, b: number) {
			this.r = r;
			this.g = g;
			this.b = b;
		}

		static random(): d3.Color {
			const color = d3.color(`rgb(${Math.random()}, ${Math.random()}, ${Math.random()})`);
			console.debug("color", color);
			return color;
		}

		static randomDark(): d3.Color {
			const h = Math.random() * 90;
			const s = Math.random() * 0.2 + 0.2;
			const v = Math.random() * 0.2;
			const color = hsv(h, s, v).rgb();
			return color;
		}

		static randomBright(): d3.Color {
			const h = Math.random() * 90 + 90;
			const s = Math.random() * 0.2 + 0.8;
			const v = Math.random() * 0.2 + 0.8;
			const color = hsv(h, s, v).rgb();
			return color;
		}

		/**
		 * @returns rgb(r*255, g*255, b*255)
		 */
		toString(): string {
			return `rgb(${this.r * 255}, ${this.g * 255}, ${this.b * 255})`;
		}
	}

	function init() {
		const canvas = document.getElementById('voronoiRoverCanvas') as HTMLCanvasElement;
		const context = canvas.getContext('2d');
		const board = new GameBoard(context, 30, 1);
		board.loop();
	}

	onMount(() => {
		init();
	});
</script>

<canvas width="1000" height="1000" id="voronoiRoverCanvas" />
