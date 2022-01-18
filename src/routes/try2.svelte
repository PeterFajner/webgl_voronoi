<script lang="typescript">
	import { hsv } from 'd3-hsv';
	import { color, Color, rgb } from 'd3-color';
	import { onMount } from 'svelte';
	import { mat4, vec2 } from 'gl-matrix';

	const NUM_FIXED_POINTS = 63;
	const NUM_SHOOTING_STARS = 1;
	const NUM_VERTICES = NUM_FIXED_POINTS + NUM_SHOOTING_STARS; // should be power of 2

	const MAX_VERTICES = 100;

	const GOAL_FPS = 60;

	class Shaders {
		static vertexShaderSource = `
        precision highp float;

        attribute lowp vec4 aVertexPosition; // position in space (corners)

		uniform mat4 uModelViewMatrix;
		uniform mat4 uProjectionMatrix;

        varying vec2 vPixelPosition;

		void main() {
			gl_Position = uProjectionMatrix * uModelViewMatrix * aVertexPosition;
            vPixelPosition = aVertexPosition.xy;
		}
        `;

		static fragmentShaderSource = `
        precision highp float;

        varying vec2 vPixelPosition;
        uniform sampler2D uVertices; // x y x y x y
        uniform sampler2D uVertexColors; // r g b r g b
        uniform int uNumVertices;

		void main() {
            // find the nearest vertex
            float sqDist;
            float closestSqDist = 100.0;
            vec3 closestColor;
            for (float i = 0.0; i < 1.0; i+=${1.0 / NUM_VERTICES}) {
                vec2 vertexPosition = texture2D(uVertices, vec2(i, 0)).xy;
                vec3 vertexColor = texture2D(uVertexColors, vec2(i, 0)).rgb;
                vec2 distance = vertexPosition - vPixelPosition;
                sqDist = pow(distance.x, 2.0) + pow(distance.y, 2.0);
                if (sqDist < closestSqDist) {
                    closestSqDist = sqDist;
                    closestColor = vertexColor;
                }
            }
            gl_FragColor = vec4(closestColor, 1);


            // this one works too - shows bubbles
            /*for (float i = 0.0; i < 1.0; i+=${1.0 / NUM_VERTICES}) {
                vec2 vertexPosition = texture2D(uVertices, vec2(i, 0)).xy;
                vec3 vertexColor = texture2D(uVertexColors, vec2(i, 0)).rgb;
                vec2 distance = vertexPosition - vPixelPosition;
                sqDist = pow(distance.x, 2.0) + pow(distance.y, 2.0);
                if (sqDist < 0.001) {
                    gl_FragColor = vec4(vertexColor, 1);
                }
            }*/

            // this one works - shows vertical bars
            //gl_FragColor = vec4(texture2D(uVertexColors, vec2(vPixelPosition.x, 0.5)).rgb, 1);
		}
        `;
	}

	class ColorTools {
		static randomColor(): Color {
			return rgb(Math.random() * 255, Math.random() * 255, Math.random() * 255);
		}

		static randomBackgroundColor(): Color {
			const h = Math.random() * 90;
			const s = Math.random() * 0.2 + 0.2;
			const v = Math.random() * 0.2;
			const color = hsv(h, s, v);
			return color;
		}

		static randomShootingStarColor(): Color {
			const h = Math.random() * 90 + 90;
			const s = Math.random() * 0.2 + 0.8;
			const v = Math.random() * 0.2 + 0.8;
			const color = hsv(h, s, v);
			return color;
		}
	}

	class ShootingStar {
		// the index of the shooting star's x coordinate in this.fixedVertices
		vertexIndex: number;
		// the index of the shooting star's r value in this.fixedVertexColors
		colorIndex: number;
		spawn: vec2; // where the shooting star is spawned
		destination: vec2; // where the shooting star despawns
		speed: number;
		game: Game;

		constructor(game: Game) {
			this.game = game;
			// register color and position and load with fake data for now
			this.colorIndex = game.fixedVertexColors.length;
			game.fixedVertexColors.push(0);
			game.fixedVertexColors.push(0);
			game.fixedVertexColors.push(0);
			game.fixedVertexColors.push(0);
			this.vertexIndex = game.fixedVertices.length;
			game.fixedVertices.push(0);
			game.fixedVertices.push(0);
			this.reset();
		}

		move(timeDeltaMs: number) {
			const travelAngle = Math.atan(
				(this.destination[1] - this.spawn[1]) / (this.destination[0] - this.spawn[0])
			);
			const travelAmount = (timeDeltaMs / 1000) * this.speed;
			const dx = travelAmount * Math.cos(travelAngle);
			const dy = travelAngle * Math.sin(travelAngle);
			this.game.fixedVertices[this.vertexIndex + 0] += dx;
			this.game.fixedVertices[this.vertexIndex + 1] += dy;
			const x = this.game.fixedVertices[this.vertexIndex + 0];
			const y = this.game.fixedVertices[this.vertexIndex + 1];
			if (x < 0 || x > 255 || y < 0 || y > 255) {
                console.debug("star despawning at", x, y);
				this.reset();
			}
		}

		reset() {
			this.speed = 50;
			// choose whether the star will travel vertically or horizontally
			if (Math.random() < 0.5) {
				// horizontally
				const left = vec2.fromValues(0, Math.random() * 255);
				const right = vec2.fromValues(255, Math.random() * 255);
				// choose whether the star will travel left->right or right->left
				if (Math.random() < 0.5) {
					this.spawn = left;
					this.destination = right;
				} else {
					this.spawn = right;
					this.destination = left;
				}
			} else {
				// vertically
				const up = vec2.fromValues(Math.random() * 255, 0);
				const down = vec2.fromValues(Math.random() * 255, 255);
				// choose whether the star will travel up->down or down->up
				if (Math.random() < 0.5) {
					this.spawn = up;
					this.destination = down;
				} else {
					this.spawn = down;
					this.destination = up;
				}
			}
			// determine star color
			const color = ColorTools.randomShootingStarColor().rgb();
			this.game.fixedVertexColors[this.colorIndex + 0] = color.r;
			this.game.fixedVertexColors[this.colorIndex + 1] = color.g;
			this.game.fixedVertexColors[this.colorIndex + 2] = color.b;
			this.game.fixedVertexColors[this.colorIndex + 3] = color.a;
			// set star position
			this.game.fixedVertices[this.vertexIndex + 0] = this.spawn[0];
			this.game.fixedVertices[this.vertexIndex + 1] = this.spawn[1];
            console.debug("star respawning at", this.spawn);
		}
	}

	class Game {
		context: WebGL2RenderingContext;
		shootingStars: ShootingStar[] = [];
		fixedVertices: number[] = []; // x y x y
		fixedVertexColors: number[] = []; // r g b a r g b a
        previousFrameTimestampMs: number = Date.now();

		constructor(context: WebGL2RenderingContext) {
			this.context = context;

			// init canvas size
			context.viewport(0, 0, context.canvas.width, context.canvas.height);

			// initialize stationary points
			for (let i = 0; i < NUM_FIXED_POINTS; i++) {
				// generate x, y in [0, 255] range
				const x = Math.random() * 255;
				const y = Math.random() * 255;
				const color = ColorTools.randomBackgroundColor().rgb();
				this.fixedVertices.push(x);
				this.fixedVertices.push(y);
				this.fixedVertexColors.push(color.r);
				this.fixedVertexColors.push(color.g);
				this.fixedVertexColors.push(color.b);
				this.fixedVertexColors.push(color.a);
			}

			// initialize shooting stars
			for (let i = 0; i < NUM_SHOOTING_STARS; i++) {
				const star = new ShootingStar(this);
                this.shootingStars.push(star);
			}

			// initialize shaders
			const vertexShader = context.createShader(context.VERTEX_SHADER);
			context.shaderSource(vertexShader, Shaders.vertexShaderSource);
			context.compileShader(vertexShader);
			if (!context.getShaderParameter(vertexShader, context.COMPILE_STATUS)) {
				console.error('Failed to compile vertex shader');
				console.error(context.getShaderInfoLog(vertexShader));
			}
			const fragmentShader = context.createShader(context.FRAGMENT_SHADER);
			context.shaderSource(fragmentShader, Shaders.fragmentShaderSource);
			context.compileShader(fragmentShader);
			if (!context.getShaderParameter(fragmentShader, context.COMPILE_STATUS)) {
				console.error('Failed to compile fragment shader');
				console.error(context.getShaderInfoLog(fragmentShader));
			}

			// create shader program
			const shaderProgram = context.createProgram();
			context.attachShader(shaderProgram, vertexShader);
			context.attachShader(shaderProgram, fragmentShader);
			context.linkProgram(shaderProgram);
			if (!context.getProgramParameter(shaderProgram, context.LINK_STATUS)) {
				console.error('Unable to initialize shader program');
				console.error(context.getProgramInfoLog(shaderProgram));
			}

			// bind GLSL variables to JS variables
			const attribLocations = {
				vertexPosition: context.getAttribLocation(shaderProgram, 'aVertexPosition')
			};
			const uniformLocations = {
				projectionMatrix: context.getUniformLocation(shaderProgram, 'uProjectionMatrix'),
				modelViewMatrix: context.getUniformLocation(shaderProgram, 'uModelViewMatrix'),
				vertices: context.getUniformLocation(shaderProgram, 'uVertices'),
				vertexColors: context.getUniformLocation(shaderProgram, 'uVertexColors'),
				numVertices: context.getUniformLocation(shaderProgram, 'uNumVertices')
			};

			// init buffers
			const vertexBuffer = context.createBuffer();
			{
				const fourCorners = [0, 0, 1, 0, 0, 1, 1, 1];
				context.bindBuffer(context.ARRAY_BUFFER, vertexBuffer);
				context.bufferData(
					context.ARRAY_BUFFER,
					new Float32Array(fourCorners),
					context.STATIC_DRAW
				);
			}

			// tell webgl how to pull positions from vertexBuffer into aVertexPosition
			{
				const numComponents = 2; // each iteration, grab x y
				const type = context.FLOAT; // 32-bit float
				const normalize = false;
				context.bindBuffer(context.ARRAY_BUFFER, vertexBuffer);
				context.vertexAttribPointer(
					attribLocations.vertexPosition,
					numComponents,
					type,
					normalize,
					0,
					0
				);
				context.enableVertexAttribArray(attribLocations.vertexPosition);
			}

			// set clear colour to be black, and clear everything on clear()
			context.clearColor(0, 0, 0, 1);
			context.clearDepth(1);

			// enable depth testing and make near things obscur far things
			context.enable(context.DEPTH_TEST);
			context.depthFunc(context.LEQUAL);

			// create camera perspective matrix
			const projectionMatrix = mat4.create();
			{
				const fieldOfView = Math.PI / 4; // 45°
				const aspectRatio = context.canvas.scrollWidth / context.canvas.scrollHeight;
				const clipNear = 0.1;
				const clipFar = 100;
				mat4.perspective(projectionMatrix, fieldOfView, aspectRatio, clipNear, clipFar);
			}

			// create a matrix to store the current drawing position
			const modelViewMatrix = mat4.create();
			mat4.translate(modelViewMatrix, modelViewMatrix, [-0.5, -0.5, -1]);

			// select drawing program
			context.useProgram(shaderProgram);

			// set shader uniforms (static each iteration)
			context.uniformMatrix4fv(uniformLocations.projectionMatrix, false, projectionMatrix);
			context.uniformMatrix4fv(uniformLocations.modelViewMatrix, false, modelViewMatrix);

			// set textures - these contain our vertices and vertex colors
			const verticesTexture = context.createTexture();
			const vertexColorsTexture = context.createTexture();
			{
				context.bindTexture(context.TEXTURE_2D, verticesTexture);
				const verticesArray = new Uint8Array(this.fixedVertices);
				context.texImage2D(
					context.TEXTURE_2D,
					0,
					context.RG8,
					NUM_VERTICES,
					1,
					0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
					context.RG,
					context.UNSIGNED_BYTE,
					verticesArray,
					0
				);
				context.generateMipmap(context.TEXTURE_2D);

				const vertexColorsArray = new Uint8Array(this.fixedVertexColors);
				context.bindTexture(context.TEXTURE_2D, vertexColorsTexture);
				context.texImage2D(
					context.TEXTURE_2D,
					0,
					context.RGBA,
					NUM_VERTICES,
					1,
					0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
					context.RGBA,
					context.UNSIGNED_BYTE,
					vertexColorsArray
				);
				context.generateMipmap(context.TEXTURE_2D);
			}

			// bind the textures to texture units
			context.activeTexture(context.TEXTURE0);
			context.bindTexture(context.TEXTURE_2D, verticesTexture);
			context.uniform1i(uniformLocations.vertices, 0);
			context.activeTexture(context.TEXTURE1);
			context.bindTexture(context.TEXTURE_2D, vertexColorsTexture);
			context.uniform1i(uniformLocations.vertexColors, 1);

			// set number of vertices
			context.uniform1i(uniformLocations.numVertices, NUM_VERTICES);

			// draw
			{
				const numVertices = 4; // four corners, defined earlier
				context.bindBuffer(context.ARRAY_BUFFER, vertexBuffer);
				context.drawArrays(context.TRIANGLE_STRIP, 0, numVertices);
			}
		}

		loop() {
            const context = this.context;

            // get frame time
            const currentFrameTimestampMs = Date.now();
			const timeElapsedMs = currentFrameTimestampMs - this.previousFrameTimestampMs;
            this.previousFrameTimestampMs = currentFrameTimestampMs;
			const fps = 1000 / timeElapsedMs;

			// update star positions
            this.shootingStars.forEach(star => {
                star.move(timeElapsedMs);
            })

            // recreate textures
			const verticesTexture = context.createTexture();
			const vertexColorsTexture = context.createTexture();
			{
				context.bindTexture(context.TEXTURE_2D, verticesTexture);
				const verticesArray = new Uint8Array(this.fixedVertices);
				context.texImage2D(
					context.TEXTURE_2D,
					0,
					context.RG8,
					NUM_VERTICES,
					1,
					0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
					context.RG,
					context.UNSIGNED_BYTE,
					verticesArray,
					0
				);
				context.generateMipmap(context.TEXTURE_2D);

				const vertexColorsArray = new Uint8Array(this.fixedVertexColors);
				context.bindTexture(context.TEXTURE_2D, vertexColorsTexture);
				context.texImage2D(
					context.TEXTURE_2D,
					0,
					context.RGBA,
					NUM_VERTICES,
					1,
					0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
					context.RGBA,
					context.UNSIGNED_BYTE,
					vertexColorsArray
				);
				context.generateMipmap(context.TEXTURE_2D);
			}

			// bind the textures to texture units
			context.activeTexture(context.TEXTURE0);
			context.bindTexture(context.TEXTURE_2D, verticesTexture);
			//context.uniform1i(uniformLocations.vertices, 0);
			context.activeTexture(context.TEXTURE1);
			context.bindTexture(context.TEXTURE_2D, vertexColorsTexture);
			//context.uniform1i(uniformLocations.vertexColors, 1);

            // draw
            {
				const numVertices = 4; // four corners, defined earlier
				context.drawArrays(context.TRIANGLE_STRIP, 0, numVertices);
			}

            setTimeout(() => this.loop(), 1000/GOAL_FPS);
		}
	}

	function init() {
		const canvas = document.getElementById('voronoiRoverCanvas') as HTMLCanvasElement;
		canvas.width = Math.max(window.innerWidth, window.innerHeight);
		canvas.height = Math.max(window.innerWidth, window.innerHeight);
		const context = canvas.getContext('webgl2');
		const game = new Game(context);
		game.loop();
	}

	onMount(() => {
		init();
	});
</script>

<canvas width="1000" height="1000" id="voronoiRoverCanvas" />
