<script type="text/typescript">
	import { Delaunay } from 'd3-delaunay';
	import { onMount } from 'svelte';
	import { xlink_attr } from 'svelte/internal';

	// the animation will aim to run at this framerate
	const TARGET_FPS = 60;

	// if the animation slows below CUTOFF_FPS for more than
	// CUTOFF_MIN_FRAMES frames, the animation will stop
	const CUTOFF_FPS = 10;
	const CUTOFF_MIN_FRAMES = 10;

	// rovers will appear and disappear this many pixels past the edge of the canvas
	const CLIP_DISTANCE = 10;

	class GameBoard {
		fixedPoints: Point[];
		rovers: Rover[];
		context: CanvasRenderingContext2D;
		colors: RGB[];
		roverIntervalSeconds: number;
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
			this.colors = [];
			this.roverIntervalSeconds = roverIntervalSeconds;

			// initialize stationary points
			this.fixedPoints = [];
			for (let i = 0; i < numFixedPoints; i++) {
				const x = Math.random() * context.canvas.clientWidth;
				const y = Math.random() * context.canvas.clientHeight;
				const color = RGB.random();
				this.fixedPoints.push(new Point(new Vector2(x, y), color));
				this.colors.push(color);
			}
		}

		loop() {
			const xmax = this.context.canvas.clientWidth;
			const ymax = this.context.canvas.clientHeight;

			// get a Delaunay triangulation of our points
			const delaunay = Delaunay.from(
				this.fixedPoints,
				(point) => point.location.x,
				(point) => point.location.y
			);

			// get the voronoi diagram, with the bounds set to the edges of the canvas
			const voronoi = delaunay.voronoi([
				0,
				0,
				this.context.canvas.clientWidth,
				this.context.canvas.clientHeight
			]);

			// render each cell
			for (let i = 0; i < this.colors.length; i++) {
				this.context.fillStyle = this.colors[i].toString();
				//voronoi.renderCell(i, this.context);
				this.context.beginPath();
				voronoi.cellPolygon(i).forEach((vertex: [x: number, y: number], index) => {
					if (index == 0) {
						this.context.moveTo(vertex[0], vertex[1]);
					} else {
						this.context.lineTo(vertex[0], vertex[1]);
					}
				});
				this.context.closePath();
				this.context.fill();
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
				console.debug('points of interest', pointsOfInterest);
				if (Math.random() < 0.5) {
					spawn = pointsOfInterest[0];
					despawn = pointsOfInterest[pointsOfInterest.length - 1];
				} else {
					spawn = pointsOfInterest[pointsOfInterest.length - 1];
					despawn = pointsOfInterest[0];
				}
				this.rovers.push(new Rover(new Vector2(spawn.x, spawn.y), spawn, despawn));
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
				// if the rover has been onscreen and is now offscreen, delete it
				if (!rover.hasEnteredScreen) {
					if (
						rover.location.x > 0 &&
						rover.location.x < xmax &&
						rover.location.y > 0 &&
						rover.location.y < ymax
					) {
						rover.hasEnteredScreen = true;
					}
				} else if (
					rover.location.x < 0 - CLIP_DISTANCE ||
					rover.location.x > xmax + CLIP_DISTANCE ||
					rover.location.y < 0 - CLIP_DISTANCE ||
					rover.location.y > ymax + CLIP_DISTANCE
				) {
					this.rovers.splice(this.rovers.indexOf(rover), 1);
				}
			});

			// render rovers
			// todo replace
			this.rovers.forEach((rover) => {
				this.context.fillStyle = rover.color.toString();
				this.context.fillRect(rover.location.x, rover.location.y, 25, 25);
			});

			// sleep
			setTimeout(() => {
				this.loop();
			}, 1000 / TARGET_FPS);
		}
	}

	class Point {
		location: Vector2;
		color: RGB;

		constructor(location: Vector2, color: RGB = null) {
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
		hasEnteredScreen = false;

		constructor(
			location: Vector2,
			spawn: Vector2,
			despawn: Vector2,
			speed: number = 100,
			color: RGB = null
		) {
			super(location, color);
			this.spawn = spawn;
			this.despawn = despawn;
			this.speed = speed;
			console.debug(`New rover heading from ${this.spawn} to ${this.despawn}`);
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

		static random(): RGB {
			return new RGB(Math.random(), Math.random(), Math.random());
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
		const board = new GameBoard(context, 30);
		board.loop();
		console.debug('Rover started');
	}

	onMount(() => {
		init();
	});
</script>

<canvas width="1000" height="1000" id="voronoiRoverCanvas" />
