<script>
	$: quad = {
		a: {x: -200, y: -200},
		b: {x: 200, y: -200},
		c: {x: 200, y: 200},
		d: {x: -200, y: 200},
	};
	$: isConvex = checkConvex(quad.a, quad.b, quad.c, quad.d);

	$: ellipse = isConvex ? quadToEllipse(quad.a,quad.b,quad.c,quad.d) : null;


	$: dragging = null;
	$: svg = null;
	$: point = svg ? svg.createSVGPoint() : null;
	$: showLabels = true
	
	function checkConvex(A,B,C,D) {
		const alpha = ((B.y - C.y)*(D.x-C.x) + (C.x-B.x)*(D.y-C.y)) / ((B.y-C.y)*(A.x-C.x) + (C.x-B.x)*(A.y-C.y));
		const beta = ((C.y - A.y)*(D.x-C.x) + (A.x-C.x)*(D.y-C.y)) / ((B.y-C.y)*(A.x-C.x) + (C.x-B.x)*(A.y-C.y));
		const gamma = 1 - alpha - beta;
		
		return alpha > 0 && beta < 0 && gamma > 0
	}
	
	function clamp(min, max, value) {
		return Math.min(Math.max(min, value), max)
	}
	
	function setVertex(name, x, y) {
		quad[name].x = clamp(-400, 400, x)
		quad[name].y = clamp(-400, 400, y)
	}
	
	function startDrag(name) {
		dragging = name;
	}
	
	function stopDrag() {
		dragging = null;
	}
	
	function moveDrag(evt) {
		if(dragging != null) {
			point.x = evt.clientX;
    	point.y = evt.clientY;
			const p =  point.matrixTransform(svg.getScreenCTM().inverse());
			setVertex(dragging, p.x, p.y)
		}
	}
	
	function quadToEllipse(W,X,Y,Z) {
		// Reconstruct matrix that transforms the unit square ((-1,-1), (1,1)) into quad (W,X,Y,Z)
		const m00 =  X.x * Y.x * Z.y - W.x * Y.x * Z.y - X.x * Y.y * Z.x + W.x * Y.y * Z.x - 
					W.x * X.y * Z.x + W.y * X.x * Z.x + W.x * X.y * Y.x - W.y * X.x * Y.x;
		const m01 =  W.x * Y.x * Z.y - W.x * X.x * Z.y - X.x * Y.y * Z.x + X.y * Y.x * Z.x - 
					W.y * Y.x * Z.x + W.y * X.x * Z.x + W.x * X.x * Y.y - W.x * X.y * Y.x;
		const m02 =  X.x * Y.x * Z.y - W.x * X.x * Z.y - W.x * Y.y * Z.x - X.y * Y.x * Z.x + 
					W.y * Y.x * Z.x + W.x * X.y * Z.x + W.x * X.x * Y.y - W.y * X.x * Y.x;
		const m10 =  X.y * Y.x * Z.y - W.y * Y.x * Z.y - W.x * X.y * Z.y + W.y * X.x * Z.y - 
					X.y * Y.y * Z.x + W.y * Y.y * Z.x + W.x * X.y * Y.y - W.y * X.x * Y.y;
		const m11 = -X.x * Y.y * Z.y + W.x * Y.y * Z.y + X.y * Y.x * Z.y - W.x * X.y * Z.y - 
					W.y * Y.y * Z.x + W.y * X.y * Z.x + W.y * X.x * Y.y - W.y * X.y * Y.x;
		const m12 =  X.x * Y.y * Z.y - W.x * Y.y * Z.y + W.y * Y.x * Z.y - W.y * X.x * Z.y - 
					X.y * Y.y * Z.x + W.y * X.y * Z.x + W.x * X.y * Y.y - W.y * X.y * Y.x;
		const m20 =  X.x * Z.y - W.x * Z.y - X.y * Z.x + W.y * Z.x - X.x * Y.y + W.x * Y.y + X.y * Y.x - W.y * Y.x;
		const m21 =  Y.x * Z.y - X.x * Z.y - Y.y * Z.x + X.y * Z.x + W.x * Y.y - W.y * Y.x - W.x * X.y + W.y * X.x;
		const m22 =  Y.x * Z.y - W.x * Z.y - Y.y * Z.x + W.y * Z.x + X.x * Y.y - X.y * Y.x + W.x * X.y - W.y * X.x;
		

		// invert matrix
		const determinant = +m00*(m11*m22-m21*m12) -m01*(m10*m22-m12*m20) +m02*(m10*m21-m11*m20);
		if(determinant == 0) return null;
		const invdet = 1/determinant; 
		const J =  (m11*m22-m21*m12)*invdet;
		const K = -(m01*m22-m02*m21)*invdet;
		const L =  (m01*m12-m02*m11)*invdet;
		const M = -(m10*m22-m12*m20)*invdet;
		const N =  (m00*m22-m02*m20)*invdet;
		const O = -(m00*m12-m10*m02)*invdet;
		const P =  (m10*m21-m20*m11)*invdet;
		const Q = -(m00*m21-m20*m01)*invdet;
		const R =  (m00*m11-m10*m01)*invdet;

		// extract ellipse coefficients from matrix
		const a = J*J + M*M - P*P;
		const b = J*K + M*N - P*Q;
		const c = K*K + N*N - Q*Q;
		const d = J*L + M*O - P*R;
		const f = K*L + N*O - Q*R;
		const g = L*L + O*O - R*R;

		// deduce ellipse center from coefficients
		const centerX = (c*d - b*f) / (b*b - a*c);
		const centerY = (a*f - b*d) / (b*b - a*c);

		// deduce ellipse radius from coefficients
		const radiusA = Math.sqrt(2*(a*f*f + c*d*d + g*b*b - 2*b*d*f - a*c*g)/((b*b - a*c) * (Math.sqrt((a-c)*(a-c) + 4*b*b) - (a+c))));
		const radiusB = Math.sqrt(2*(a*f*f + c*d*d + g*b*b - 2*b*d*f - a*c*g)/((b*b - a*c) * (-Math.sqrt((a-c)*(a-c) + 4*b*b) - (a+c))));

		// deduce ellipse rotation from coefficients
		let angle = 0;
		if(b==0 && a <= c) {
				angle = 0;
		} else if(b == 0 && a >= c) {
				angle = Math.PI / 2;
		} else if(b != 0 && a > c) {
				angle = Math.PI / 2 + 0.5 * (Math.PI / 2 - Math.atan2((a-c), (2*b)));
		} else if(b != 0 && a <= c) {
				angle = Math.PI / 2 + 0.5 * (Math.PI / 2 - Math.atan2((a-c), (2*b)));
		}
				
		return {
			centerX, centerY, radiusA, radiusB, angle
		}
	}
</script>

<style>
	:global(body) {
		margin: 0;
		box-sizing: border-box;
	}

	svg {
		display: block;
		height: auto;
		border: 1px solid gray;
		flex-shrink: 0;
		flex-grow: 1;
		width: 100%;
		box-sizing: border-box;
	}
	pre {
		display: block;
		overflow-x: scroll;
		flex-shrink: 10;
		flex-basis: 10%;
		flex-grow: 1;
		width: 100%;
		box-sizing: border-box;
		background: #222;
		color: #fff;
		padding: 1em;
		scroll-padding: 1em;
	}
	
	.container {
		display: flex;
		flex-wrap: wrap;
		flex-direction: row;
		gap: 3em;
		padding: 1em;
		box-sizing: border-box;
	}
	.col {
		flex-basis: 20em;
		flex-grow: 1;
		flex-shrink: 1;
		width: 100%;
		box-sizing: border-box;
		max-width: 60em;
	}
	
	.quad { fill: #ddd; }
	.corner { fill: #00cccc; stroke: white; stroke-width: 3pt; cursor: move;  }
	.mirrored { fill: #fff; stroke-dasharray: 7 7; stroke: #00cccc; stroke-width: 4pt;   }
	.error {
		fill: #ff7777;
		stroke: #ff0000;
		color: #ff0000;
		stroke-width: 3pt;
	}
	.center {fill: white;}
	.label-bg {fill:black; stroke: black;font-size: 15pt;stroke:2pt;}
	.label {fill:white; font-size: 15pt;}
	.cross {stroke: white;stroke-width:2pt;}

	h1 {
		margin: 0;
	}
</style>
<svelte:window on:pointerup={stopDrag} on:pointermove={(e) => moveDrag(e)} />

<div class="container">

<div class="col">
	
<h1>
	2D Convex Quadriliteral to Ellipse Conversion 
</h1>
	

	{#if !isConvex}
	<div class="error">
		Quadriliteral is not convex. Drag the vertices to create a convex shape.
	</div>
	{:else}
	{#if dragging}
	<div>
		Dragging vertex {dragging}
	</div>
	{:else}
	<div>
		Drag the points to change the shape.
	</div>
	{/if}
	{/if}
	
	
	<fieldset>
		<legend>Options</legend>

		<label><input type="checkbox" bind:checked={showLabels} /> Show Labels</label>

	</fieldset>
<svg bind:this={svg} viewBox="-500 -500 1000 1000">
	<polygon class="quad" class:error="{!isConvex}" points="{quad.a.x} {quad.a.y} {quad.b.x} {quad.b.y} {quad.c.x} {quad.c.y} {quad.d.x} {quad.d.y}"></polygon>
	
	{#if ellipse}
	<ellipse cx={ellipse.centerX} cy={ellipse.centerY} rx={ellipse.radiusA} ry={ellipse.radiusB} transform="rotate({ellipse.angle* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})" />
	
	{#if showLabels}
	<circle cx={ellipse.centerX} cy={ellipse.centerY} r="5" class="center"></circle>
	<line class="cross" 
				x1={ellipse.centerX - ellipse.radiusA} 
				y1={ellipse.centerY} 
				x2={ellipse.centerX + ellipse.radiusA} 
				y2={ellipse.centerY} 
				transform="rotate({ellipse.angle* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})"></line>
	<line class="cross"
				x1={ellipse.centerX} 
				y1={ellipse.centerY - ellipse.radiusB} 
				x2={ellipse.centerX} 
				y2={ellipse.centerY + ellipse.radiusB} 
				transform="rotate({ellipse.angle* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})"></line>
<text class="label-bg" x={ellipse.centerX + 20} y={ellipse.centerY - 10}
				transform="rotate({-90 + Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Center: ({Math.round(ellipse.centerX)}, {Math.round(ellipse.centerY)})
</text>
	<text class="label-bg"  x={ellipse.centerX + 20} y={ellipse.centerY - 10} 
				transform="rotate({180 + Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Radius: ({Math.round(ellipse.radiusA)}, {Math.round(ellipse.radiusB)}) 
</text>
	<text class="label-bg"  x={ellipse.centerX + 20} y={ellipse.centerY - 10}
				transform="rotate({90+Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Angle: {Math.round(ellipse.angle * 180/Math.PI)}deg
</text>
	<text class="label" x={ellipse.centerX + 20} y={ellipse.centerY - 10}
				transform="rotate({-90 + Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Center: ({Math.round(ellipse.centerX)}, {Math.round(ellipse.centerY)})
</text>
	<text class="label"  x={ellipse.centerX + 20} y={ellipse.centerY - 10} 
				transform="rotate({180 + Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Radius: ({Math.round(ellipse.radiusA)}, {Math.round(ellipse.radiusB)}) 
</text>
	<text class="label"  x={ellipse.centerX + 20} y={ellipse.centerY - 10}
				transform="rotate({90+Math.abs(ellipse.angle)* (180/Math.PI)}, {ellipse.centerX} {ellipse.centerY})">
Angle: {Math.round(ellipse.angle * 180/Math.PI)}deg
</text>
	{/if}
	{/if}
	
	<circle class="corner" on:pointerdown={() => startDrag("a")} cx={quad.a.x} cy={quad.a.y} r="20"></circle>
	<circle class="corner" on:pointerdown={() => startDrag('b')} cx={quad.b.x} cy={quad.b.y} r="20"></circle>
	<circle class="corner" on:pointerdown={() => startDrag("c")} cx={quad.c.x} cy={quad.c.y} r="20"></circle>
	<circle class="corner" on:pointerdown={() => startDrag("d")} cx={quad.d.x} cy={quad.d.y} r="20"></circle>

</svg>
<h2>Context</h2>
	<p>
		A 3-dimensional scene can be rendered by projecting the geometry from 3D space onto a 3-dimensional plane. The proportion of the geometry may be distorted in the process to achieve a perspective effect. Especially circles are being squished into ellipses when being projected. But the exact relation between the size and position of a circle and 3D space and the resulting size and position of the projected ellipse is not that straight-forward as one might think.
	</p>
	<p>
		So when rendering a 3D scene containg circular shapes one could simply first discretizing the circle into line segments and then projecting only theses segments with is easily done by just transforming the end-points of each segment. But discretizing smooth shapes to early might reduce later image quality.
	</p>
	<p>
		So the better alternative might be to project the smooth circular shape from 3-dimensions into the correct smooth 2-dimensional ellipse. Christopher Brierley Jones <a href="http://chrisjones.id.au/Ellipses/ellipse.html">describes the process on his website</a>. The idea is instead of projecting the circle, to project it's bounding rectangle and then find the matrix that transforms that resulting quadriliteral into a unit square. Then extract the parameters of the ellipse from this matrix. The key insight is that for each convex quadriliteral is exactly one largest ellipse that fits it perfectly (by being tangential on all 4 sides). And the perspective projection of the bounding rectangle is always a convex quadriliteral.
	</p>
	<p>
		But I could not find any  showcase implementation or ready to use code of this approach online so I implemented it myself according to Christopher Brierley Jones notes. So feel free to play around by morphing the circle above or use my implementation. My implementation is licensed under <a href="https://mit-license.org/">MIT</a>
	</p>
</div>
	
<div class="col">
<h2>Implementation</h2>
According to <a href="http://chrisjones.id.au/Ellipses/ellipse.html"><cite href="http://chrisjones.id.au/Ellipses/ellipse.html">http://chrisjones.id.au/Ellipses/ellipse.html</cite></a>
<pre>
{`function quadToEllipse(W,X,Y,Z) {
	// Reconstruct matrix that transforms the unit square ((-1,-1), (1,1)) into quad (W,X,Y,Z)
	const m00 =  X.x * Y.x * Z.y - W.x * Y.x * Z.y - X.x * Y.y * Z.x + W.x * Y.y * Z.x - 
				W.x * X.y * Z.x + W.y * X.x * Z.x + W.x * X.y * Y.x - W.y * X.x * Y.x;
	const m01 =  W.x * Y.x * Z.y - W.x * X.x * Z.y - X.x * Y.y * Z.x + X.y * Y.x * Z.x - 
				W.y * Y.x * Z.x + W.y * X.x * Z.x + W.x * X.x * Y.y - W.x * X.y * Y.x;
	const m02 =  X.x * Y.x * Z.y - W.x * X.x * Z.y - W.x * Y.y * Z.x - X.y * Y.x * Z.x + 
				W.y * Y.x * Z.x + W.x * X.y * Z.x + W.x * X.x * Y.y - W.y * X.x * Y.x;
	const m10 =  X.y * Y.x * Z.y - W.y * Y.x * Z.y - W.x * X.y * Z.y + W.y * X.x * Z.y - 
				X.y * Y.y * Z.x + W.y * Y.y * Z.x + W.x * X.y * Y.y - W.y * X.x * Y.y;
	const m11 = -X.x * Y.y * Z.y + W.x * Y.y * Z.y + X.y * Y.x * Z.y - W.x * X.y * Z.y - 
				W.y * Y.y * Z.x + W.y * X.y * Z.x + W.y * X.x * Y.y - W.y * X.y * Y.x;
	const m12 =  X.x * Y.y * Z.y - W.x * Y.y * Z.y + W.y * Y.x * Z.y - W.y * X.x * Z.y - 
				X.y * Y.y * Z.x + W.y * X.y * Z.x + W.x * X.y * Y.y - W.y * X.y * Y.x;
	const m20 =  X.x * Z.y - W.x * Z.y - X.y * Z.x + W.y * Z.x - X.x * Y.y + W.x * Y.y + X.y * Y.x - W.y * Y.x;
	const m21 =  Y.x * Z.y - X.x * Z.y - Y.y * Z.x + X.y * Z.x + W.x * Y.y - W.y * Y.x - W.x * X.y + W.y * X.x;
	const m22 =  Y.x * Z.y - W.x * Z.y - Y.y * Z.x + W.y * Z.x + X.x * Y.y - X.y * Y.x + W.x * X.y - W.y * X.x;
	

	// invert matrix
	const determinant = +m00*(m11*m22-m21*m12) -m01*(m10*m22-m12*m20) +m02*(m10*m21-m11*m20);
	
	if(determinant == 0) return null;
	
	const invdet = 1 / determinant; 
	const J =  (m11*m22 - m21*m12) * invdet;
	const K = -(m01*m22 - m02*m21) * invdet;
	const L =  (m01*m12 - m02*m11) * invdet;
	const M = -(m10*m22 - m12*m20) * invdet;
	const N =  (m00*m22 - m02*m20) * invdet;
	const O = -(m00*m12 - m10*m02) * invdet;
	const P =  (m10*m21 - m20*m11) * invdet;
	const Q = -(m00*m21 - m20*m01) * invdet;
	const R =  (m00*m11 - m10*m01) * invdet;

	// extract ellipse coefficients from matrix
	const a = J*J + M*M - P*P;
	const b = J*K + M*N - P*Q;
	const c = K*K + N*N - Q*Q;
	const d = J*L + M*O - P*R;
	const f = K*L + N*O - Q*R;
	const g = L*L + O*O - R*R;

	// deduce ellipse center from coefficients
	const centerX = (c*d - b*f) / (b*b - a*c);
	const centerY = (a*f - b*d) / (b*b - a*c);

	// deduce ellipse radius from coefficients
	const radiusA = Math.sqrt(2*(a*f*f + c*d*d + g*b*b - 2*b*d*f - a*c*g)/((b*b - a*c) * 
		(Math.sqrt((a-c)*(a-c) + 4*b*b) - (a+c))));
	const radiusB = Math.sqrt(2*(a*f*f + c*d*d + g*b*b - 2*b*d*f - a*c*g)/((b*b - a*c) * 
		(-Math.sqrt((a-c)*(a-c) + 4*b*b) - (a+c))));

	// deduce ellipse rotation from coefficients
	let angle = 0;
	if(b==0 && a <= c) {
			angle = 0;
	} else if(b == 0 && a >= c) {
			angle = Math.PI / 2;
	} else if(b != 0 && a > c) {
			angle = Math.PI / 2 + 0.5 * (Math.PI / 2 - Math.atan2((a-c), (2*b)));
	} else if(b != 0 && a <= c) {
			angle = Math.PI / 2 + 0.5 * (Math.PI / 2 - Math.atan2((a-c), (2*b)));
	}
			
	return {
		centerX, centerY, radiusA, radiusB, angle
	}
}`}
</pre>
		</div>
		</div>