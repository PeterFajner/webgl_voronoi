<script lang="typescript">
	import { hsv } from 'd3-hsv';
	import type { Color, rgb } from 'd3-color';
	import { onMount } from 'svelte';
	import { mat4 } from 'gl-matrix';

	const NUM_FIXED_POINTS = 10;
    const NUM_SHOOTING_STARS = 0;
    const NUM_VERTICES = NUM_FIXED_POINTS + NUM_SHOOTING_STARS;

	const MAX_VERTICES = 100;

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
            float nearestVertexSqDist = -1.0;
            float sqDist;
            vec3 nearestVertexColor = vec3(0, 1, 0);
            for (int i = 0; i < ${MAX_VERTICES}; i++) {
                if (i >= uNumVertices) {
                    break;
                }
                vec2 vertexPosition = texture2D(uVertices, vec2(i, 0.5)).xy;
                vec3 vertexColor = texture2D(uVertexColors, vec2(i, 0.5)).rgb;
                vec2 distance = vertexPosition - vPixelPosition;
                sqDist = pow(distance.x, 2.0) + pow(distance.y, 2.0);
                if (nearestVertexSqDist < 0.0 || sqDist < nearestVertexSqDist) {
                    nearestVertexSqDist = sqDist;
                    nearestVertexColor = vertexColor;
                }
            }
			gl_FragColor = vec4(nearestVertexColor, 1);

            // testing
            /*if (sqDist < 0.2) {
                gl_FragColor = vec4(0, 0, 1, 1);
            } else {
                gl_FragColor = vec4(0, 1, 0, 1);
            }*/
            //gl_FragColor = vec4(0, vPixelPosition, 1);

            /*vec2 vertexPosition = texture2D(uVertices, vec2(3, 0)).xy;
            float threshold = 0.01;
            if (abs(vPixelPosition.x - vertexPosition.x) < threshold && abs(vPixelPosition.y - vertexPosition.y) < threshold) {
                gl_FragColor = vec4(0, 0, 1, 1);
            } else {
                gl_FragColor = vec4(0, 1, 0, 1);
            }*/

            gl_FragColor = vec4(texture2D(uVertexColors, vPixelPosition).rgb, 1);
		}
        `;
	}

	class ColorTools {
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

	class Game {
		context: WebGL2RenderingContext;

		constructor(context: WebGL2RenderingContext) {
			this.context = context;
            let fixedVertices: number[] = []; // x y x y x y
            let fixedVertexColors: number[] = []; // r g b r g b r g b

			// init canvas size
			context.viewport(0, 0, context.canvas.width, context.canvas.height);

			// initialize stationary points
			for (let i = 0; i < NUM_FIXED_POINTS; i++) {
				// generate x, y in [-1, 1] range
				const x = Math.random() * 2 - 1;
				const y = Math.random() * 2 - 1;
				const color = ColorTools.randomBackgroundColor().rgb();
				fixedVertices.push(x);
				fixedVertices.push(y);
				fixedVertexColors.push(color.r/255);
				fixedVertexColors.push(color.g/255);
				fixedVertexColors.push(color.b/255);
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
                numVertices: context.getUniformLocation(shaderProgram, 'uNumVertices'),
			};

			// init buffers
			const vertexBuffer = context.createBuffer();
			{
				const fourCorners = [-1, -1, 1, -1, -1, 1, 1, 1];
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
			mat4.translate(modelViewMatrix, modelViewMatrix, [-0.0, 0.0, -2.8]);

			// select drawing program
			context.useProgram(shaderProgram);

			// set shader uniforms (static each iteration)
			context.uniformMatrix4fv(uniformLocations.projectionMatrix, false, projectionMatrix);
			context.uniformMatrix4fv(uniformLocations.modelViewMatrix, false, modelViewMatrix);

			// set textures - these contain our vertices and vertex colors
            const verticesTexture = context.createTexture();
            const vertexColorsTexture = context.createTexture();
			{
                const verticesArray = new Float32Array(fixedVertices);
                context.bindTexture(context.TEXTURE_2D, verticesTexture);
                context.texImage2D(
                    context.TEXTURE_2D,
                    0,
                    context.RG32F,
                    NUM_VERTICES,
                    1,
                    0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
                    context.RG,
                    context.FLOAT,
                    verticesArray,
                    0
                );
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_MIN_FILTER, context.LINEAR);
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_WRAP_S, context.CLAMP_TO_EDGE);
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_WRAP_R, context.CLAMP_TO_EDGE);

                const vertexColorsArray = new Float32Array(fixedVertexColors);
                context.bindTexture(context.TEXTURE_2D, vertexColorsTexture);
                context.texImage2D(
                    context.TEXTURE_2D,
                    0,
                    context.RGB32F,
                    NUM_VERTICES,
                    1,
                    0,
					// must be this according to Table 2 @ https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexImage2D.xhtml
                    context.RGB,
                    context.FLOAT,
                    vertexColorsArray,
                    0
                );
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_MIN_FILTER, context.LINEAR);
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_WRAP_S, context.CLAMP_TO_EDGE);
                context.texParameteri(context.TEXTURE_2D, context.TEXTURE_WRAP_R, context.CLAMP_TO_EDGE);
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
				context.drawArrays(context.TRIANGLE_STRIP, 0, 4);
			}
		}

		loop() {
			// todo run interpolation thread
			// todo run rendering thread
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
