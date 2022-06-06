<template>
    <div id="canvas"></div>
</template>

<script>
    import * as THREE from 'three'

	import {
		EventDispatcher,
		MOUSE,
		Quaternion,
		Vector2,
		Vector3
	} from 'three';

    export default {
        mounted() {
            const _changeEvent = { type: 'change' };
const _startEvent = { type: 'start' };
const _endEvent = { type: 'end' };

class TrackballControls extends EventDispatcher {

	constructor( object, domElement ) {

		super();

		if ( domElement === undefined ) console.warn( 'THREE.TrackballControls: The second parameter "domElement" is now mandatory.' );
		if ( domElement === document ) console.error( 'THREE.TrackballControls: "document" should not be used as the target "domElement". Please use "renderer.domElement" instead.' );

		const scope = this;
		const STATE = { NONE: - 1, ROTATE: 0, ZOOM: 1, PAN: 2, TOUCH_ROTATE: 3, TOUCH_ZOOM_PAN: 4 };

		this.object = object;
		this.domElement = domElement;
		this.domElement.style.touchAction = 'none'; // disable touch scroll

		// API

		this.enabled = true;

		this.screen = { left: 0, top: 0, width: 0, height: 0 };

		this.rotateSpeed = 1.0;
		this.zoomSpeed = 1.2;
		this.panSpeed = 0.3;

		this.noRotate = false;
		this.noZoom = false;
		this.noPan = false;

		this.staticMoving = false;
		this.dynamicDampingFactor = 0.2;

		this.minDistance = 0;
		this.maxDistance = Infinity;

		this.keys = [ 'KeyA' /*A*/, 'KeyS' /*S*/, 'KeyD' /*D*/ ];

		this.mouseButtons = { LEFT: MOUSE.ROTATE, MIDDLE: MOUSE.DOLLY, RIGHT: MOUSE.PAN };

		// internals

		this.target = new Vector3();

		const EPS = 0.000001;

		const lastPosition = new Vector3();
		let lastZoom = 1;

		let _state = STATE.NONE,
			_keyState = STATE.NONE,

			_touchZoomDistanceStart = 0,
			_touchZoomDistanceEnd = 0,

			_lastAngle = 0;

		const _eye = new Vector3(),

			_movePrev = new Vector2(),
			_moveCurr = new Vector2(),

			_lastAxis = new Vector3(),

			_zoomStart = new Vector2(),
			_zoomEnd = new Vector2(),

			_panStart = new Vector2(),
			_panEnd = new Vector2(),

			_pointers = [],
			_pointerPositions = {};

		// for reset

		this.target0 = this.target.clone();
		this.position0 = this.object.position.clone();
		this.up0 = this.object.up.clone();
		this.zoom0 = this.object.zoom;

		// methods

		this.handleResize = function () {

			const box = scope.domElement.getBoundingClientRect();
			// adjustments come from similar code in the jquery offset() function
			const d = scope.domElement.ownerDocument.documentElement;
			scope.screen.left = box.left + window.pageXOffset - d.clientLeft;
			scope.screen.top = box.top + window.pageYOffset - d.clientTop;
			scope.screen.width = box.width;
			scope.screen.height = box.height;

		};

		const getMouseOnScreen = ( function () {

			const vector = new Vector2();

			return function getMouseOnScreen( pageX, pageY ) {

				vector.set(
					( pageX - scope.screen.left ) / scope.screen.width,
					( pageY - scope.screen.top ) / scope.screen.height
				);

				return vector;

			};

		}() );

		const getMouseOnCircle = ( function () {

			const vector = new Vector2();

			return function getMouseOnCircle( pageX, pageY ) {

				vector.set(
					( ( pageX - scope.screen.width * 0.5 - scope.screen.left ) / ( scope.screen.width * 0.5 ) ),
					( ( scope.screen.height + 2 * ( scope.screen.top - pageY ) ) / scope.screen.width ) // screen.width intentional
				);

				return vector;

			};

		}() );

		this.rotateCamera = ( function () {

			const axis = new Vector3(),
				quaternion = new Quaternion(),
				eyeDirection = new Vector3(),
				objectUpDirection = new Vector3(),
				objectSidewaysDirection = new Vector3(),
				moveDirection = new Vector3();

			return function rotateCamera() {

				moveDirection.set( _moveCurr.x - _movePrev.x, _moveCurr.y - _movePrev.y, 0 );
				let angle = moveDirection.length();

				if ( angle ) {

					_eye.copy( scope.object.position ).sub( scope.target );

					eyeDirection.copy( _eye ).normalize();
					objectUpDirection.copy( scope.object.up ).normalize();
					objectSidewaysDirection.crossVectors( objectUpDirection, eyeDirection ).normalize();

					objectUpDirection.setLength( _moveCurr.y - _movePrev.y );
					objectSidewaysDirection.setLength( _moveCurr.x - _movePrev.x );

					moveDirection.copy( objectUpDirection.add( objectSidewaysDirection ) );

					axis.crossVectors( moveDirection, _eye ).normalize();

					angle *= scope.rotateSpeed;
					quaternion.setFromAxisAngle( axis, angle );

					_eye.applyQuaternion( quaternion );
					scope.object.up.applyQuaternion( quaternion );

					_lastAxis.copy( axis );
					_lastAngle = angle;

				} else if ( ! scope.staticMoving && _lastAngle ) {

					_lastAngle *= Math.sqrt( 1.0 - scope.dynamicDampingFactor );
					_eye.copy( scope.object.position ).sub( scope.target );
					quaternion.setFromAxisAngle( _lastAxis, _lastAngle );
					_eye.applyQuaternion( quaternion );
					scope.object.up.applyQuaternion( quaternion );

				}

				_movePrev.copy( _moveCurr );

			};

		}() );


		this.zoomCamera = function () {

			let factor;

			if ( _state === STATE.TOUCH_ZOOM_PAN ) {

				factor = _touchZoomDistanceStart / _touchZoomDistanceEnd;
				_touchZoomDistanceStart = _touchZoomDistanceEnd;

				if ( scope.object.isPerspectiveCamera ) {

					_eye.multiplyScalar( factor );

				} else if ( scope.object.isOrthographicCamera ) {

					scope.object.zoom /= factor;
					scope.object.updateProjectionMatrix();

				} else {

					console.warn( 'THREE.TrackballControls: Unsupported camera type' );

				}

			} else {

				factor = 1.0 + ( _zoomEnd.y - _zoomStart.y ) * scope.zoomSpeed;

				if ( factor !== 1.0 && factor > 0.0 ) {

					if ( scope.object.isPerspectiveCamera ) {

						_eye.multiplyScalar( factor );

					} else if ( scope.object.isOrthographicCamera ) {

						scope.object.zoom /= factor;
						scope.object.updateProjectionMatrix();

					} else {

						console.warn( 'THREE.TrackballControls: Unsupported camera type' );

					}

				}

				if ( scope.staticMoving ) {

					_zoomStart.copy( _zoomEnd );

				} else {

					_zoomStart.y += ( _zoomEnd.y - _zoomStart.y ) * this.dynamicDampingFactor;

				}

			}

		};

		this.panCamera = ( function () {

			const mouseChange = new Vector2(),
				objectUp = new Vector3(),
				pan = new Vector3();

			return function panCamera() {

				mouseChange.copy( _panEnd ).sub( _panStart );

				if ( mouseChange.lengthSq() ) {

					if ( scope.object.isOrthographicCamera ) {

						const scale_x = ( scope.object.right - scope.object.left ) / scope.object.zoom / scope.domElement.clientWidth;
						const scale_y = ( scope.object.top - scope.object.bottom ) / scope.object.zoom / scope.domElement.clientWidth;

						mouseChange.x *= scale_x;
						mouseChange.y *= scale_y;

					}

					mouseChange.multiplyScalar( _eye.length() * scope.panSpeed );

					pan.copy( _eye ).cross( scope.object.up ).setLength( mouseChange.x );
					pan.add( objectUp.copy( scope.object.up ).setLength( mouseChange.y ) );

					scope.object.position.add( pan );
					scope.target.add( pan );

					if ( scope.staticMoving ) {

						_panStart.copy( _panEnd );

					} else {

						_panStart.add( mouseChange.subVectors( _panEnd, _panStart ).multiplyScalar( scope.dynamicDampingFactor ) );

					}

				}

			};

		}() );

		this.checkDistances = function () {

			if ( ! scope.noZoom || ! scope.noPan ) {

				if ( _eye.lengthSq() > scope.maxDistance * scope.maxDistance ) {

					scope.object.position.addVectors( scope.target, _eye.setLength( scope.maxDistance ) );
					_zoomStart.copy( _zoomEnd );

				}

				if ( _eye.lengthSq() < scope.minDistance * scope.minDistance ) {

					scope.object.position.addVectors( scope.target, _eye.setLength( scope.minDistance ) );
					_zoomStart.copy( _zoomEnd );

				}

			}

		};

		this.update = function () {

			_eye.subVectors( scope.object.position, scope.target );

			if ( ! scope.noRotate ) {

				scope.rotateCamera();

			}

			if ( ! scope.noZoom ) {

				scope.zoomCamera();

			}

			if ( ! scope.noPan ) {

				scope.panCamera();

			}

			scope.object.position.addVectors( scope.target, _eye );

			if ( scope.object.isPerspectiveCamera ) {

				scope.checkDistances();

				scope.object.lookAt( scope.target );

				if ( lastPosition.distanceToSquared( scope.object.position ) > EPS ) {

					scope.dispatchEvent( _changeEvent );

					lastPosition.copy( scope.object.position );

				}

			} else if ( scope.object.isOrthographicCamera ) {

				scope.object.lookAt( scope.target );

				if ( lastPosition.distanceToSquared( scope.object.position ) > EPS || lastZoom !== scope.object.zoom ) {

					scope.dispatchEvent( _changeEvent );

					lastPosition.copy( scope.object.position );
					lastZoom = scope.object.zoom;

				}

			} else {

				console.warn( 'THREE.TrackballControls: Unsupported camera type' );

			}

		};

		this.reset = function () {

			_state = STATE.NONE;
			_keyState = STATE.NONE;

			scope.target.copy( scope.target0 );
			scope.object.position.copy( scope.position0 );
			scope.object.up.copy( scope.up0 );
			scope.object.zoom = scope.zoom0;

			scope.object.updateProjectionMatrix();

			_eye.subVectors( scope.object.position, scope.target );

			scope.object.lookAt( scope.target );

			scope.dispatchEvent( _changeEvent );

			lastPosition.copy( scope.object.position );
			lastZoom = scope.object.zoom;

		};

		// listeners

		function onPointerDown( event ) {

			if ( scope.enabled === false ) return;

			if ( _pointers.length === 0 ) {

				scope.domElement.setPointerCapture( event.pointerId );

				scope.domElement.addEventListener( 'pointermove', onPointerMove );
				scope.domElement.addEventListener( 'pointerup', onPointerUp );

			}

			//

			addPointer( event );

			if ( event.pointerType === 'touch' ) {

				onTouchStart( event );

			} else {

				onMouseDown( event );

			}

		}

		function onPointerMove( event ) {

			if ( scope.enabled === false ) return;

			if ( event.pointerType === 'touch' ) {

				onTouchMove( event );

			} else {

				onMouseMove( event );

			}

		}

		function onPointerUp( event ) {

			if ( scope.enabled === false ) return;

			if ( event.pointerType === 'touch' ) {

				onTouchEnd( event );

			} else {

				onMouseUp();

			}

			//

			removePointer( event );

			if ( _pointers.length === 0 ) {

				scope.domElement.releasePointerCapture( event.pointerId );

				scope.domElement.removeEventListener( 'pointermove', onPointerMove );
				scope.domElement.removeEventListener( 'pointerup', onPointerUp );

			}


		}

		function onPointerCancel( event ) {

			removePointer( event );

		}

		function keydown( event ) {

			if ( scope.enabled === false ) return;

			window.removeEventListener( 'keydown', keydown );

			if ( _keyState !== STATE.NONE ) {

				return;

			} else if ( event.code === scope.keys[ STATE.ROTATE ] && ! scope.noRotate ) {

				_keyState = STATE.ROTATE;

			} else if ( event.code === scope.keys[ STATE.ZOOM ] && ! scope.noZoom ) {

				_keyState = STATE.ZOOM;

			} else if ( event.code === scope.keys[ STATE.PAN ] && ! scope.noPan ) {

				_keyState = STATE.PAN;

			}

		}

		function keyup() {

			if ( scope.enabled === false ) return;

			_keyState = STATE.NONE;

			window.addEventListener( 'keydown', keydown );

		}

		function onMouseDown( event ) {

			if ( _state === STATE.NONE ) {

				switch ( event.button ) {

					case scope.mouseButtons.LEFT:
						_state = STATE.ROTATE;
						break;

					case scope.mouseButtons.MIDDLE:
						_state = STATE.ZOOM;
						break;

					case scope.mouseButtons.RIGHT:
						_state = STATE.PAN;
						break;

				}

			}

			const state = ( _keyState !== STATE.NONE ) ? _keyState : _state;

			if ( state === STATE.ROTATE && ! scope.noRotate ) {

				_moveCurr.copy( getMouseOnCircle( event.pageX, event.pageY ) );
				_movePrev.copy( _moveCurr );

			} else if ( state === STATE.ZOOM && ! scope.noZoom ) {

				_zoomStart.copy( getMouseOnScreen( event.pageX, event.pageY ) );
				_zoomEnd.copy( _zoomStart );

			} else if ( state === STATE.PAN && ! scope.noPan ) {

				_panStart.copy( getMouseOnScreen( event.pageX, event.pageY ) );
				_panEnd.copy( _panStart );

			}

			scope.dispatchEvent( _startEvent );

		}

		function onMouseMove( event ) {

			const state = ( _keyState !== STATE.NONE ) ? _keyState : _state;

			if ( state === STATE.ROTATE && ! scope.noRotate ) {

				_movePrev.copy( _moveCurr );
				_moveCurr.copy( getMouseOnCircle( event.pageX, event.pageY ) );

			} else if ( state === STATE.ZOOM && ! scope.noZoom ) {

				_zoomEnd.copy( getMouseOnScreen( event.pageX, event.pageY ) );

			} else if ( state === STATE.PAN && ! scope.noPan ) {

				_panEnd.copy( getMouseOnScreen( event.pageX, event.pageY ) );

			}

		}

		function onMouseUp() {

			_state = STATE.NONE;

			scope.dispatchEvent( _endEvent );

		}

		function onMouseWheel( event ) {

			if ( scope.enabled === false ) return;

			if ( scope.noZoom === true ) return;

			event.preventDefault();

			switch ( event.deltaMode ) {

				case 2:
					// Zoom in pages
					_zoomStart.y -= event.deltaY * 0.025;
					break;

				case 1:
					// Zoom in lines
					_zoomStart.y -= event.deltaY * 0.01;
					break;

				default:
					// undefined, 0, assume pixels
					_zoomStart.y -= event.deltaY * 0.00025;
					break;

			}

			scope.dispatchEvent( _startEvent );
			scope.dispatchEvent( _endEvent );

		}

		function onTouchStart( event ) {

			trackPointer( event );

			switch ( _pointers.length ) {

				case 1:
					_state = STATE.TOUCH_ROTATE;
					_moveCurr.copy( getMouseOnCircle( _pointers[ 0 ].pageX, _pointers[ 0 ].pageY ) );
					_movePrev.copy( _moveCurr );
					break;

				default: // 2 or more
					_state = STATE.TOUCH_ZOOM_PAN;
					const dx = _pointers[ 0 ].pageX - _pointers[ 1 ].pageX;
					const dy = _pointers[ 0 ].pageY - _pointers[ 1 ].pageY;
					_touchZoomDistanceEnd = _touchZoomDistanceStart = Math.sqrt( dx * dx + dy * dy );

					const x = ( _pointers[ 0 ].pageX + _pointers[ 1 ].pageX ) / 2;
					const y = ( _pointers[ 0 ].pageY + _pointers[ 1 ].pageY ) / 2;
					_panStart.copy( getMouseOnScreen( x, y ) );
					_panEnd.copy( _panStart );
					break;

			}

			scope.dispatchEvent( _startEvent );

		}

		function onTouchMove( event ) {

			trackPointer( event );

			switch ( _pointers.length ) {

				case 1:
					_movePrev.copy( _moveCurr );
					_moveCurr.copy( getMouseOnCircle( event.pageX, event.pageY ) );
					break;

				default: // 2 or more

					const position = getSecondPointerPosition( event );

					const dx = event.pageX - position.x;
					const dy = event.pageY - position.y;
					_touchZoomDistanceEnd = Math.sqrt( dx * dx + dy * dy );

					const x = ( event.pageX + position.x ) / 2;
					const y = ( event.pageY + position.y ) / 2;
					_panEnd.copy( getMouseOnScreen( x, y ) );
					break;

			}

		}

		function onTouchEnd( event ) {

			switch ( _pointers.length ) {

				case 0:
					_state = STATE.NONE;
					break;

				case 1:
					_state = STATE.TOUCH_ROTATE;
					_moveCurr.copy( getMouseOnCircle( event.pageX, event.pageY ) );
					_movePrev.copy( _moveCurr );
					break;

				case 2:
					_state = STATE.TOUCH_ZOOM_PAN;
					_moveCurr.copy( getMouseOnCircle( event.pageX - _movePrev.x, event.pageY - _movePrev.y ) );
					_movePrev.copy( _moveCurr );
					break;

			}

			scope.dispatchEvent( _endEvent );

		}

		function contextmenu( event ) {

			if ( scope.enabled === false ) return;

			event.preventDefault();

		}

		function addPointer( event ) {

			_pointers.push( event );

		}

		function removePointer( event ) {

			delete _pointerPositions[ event.pointerId ];

			for ( let i = 0; i < _pointers.length; i ++ ) {

				if ( _pointers[ i ].pointerId == event.pointerId ) {

					_pointers.splice( i, 1 );
					return;

				}

			}

		}

		function trackPointer( event ) {

			let position = _pointerPositions[ event.pointerId ];

			if ( position === undefined ) {

				position = new Vector2();
				_pointerPositions[ event.pointerId ] = position;

			}

			position.set( event.pageX, event.pageY );

		}

		function getSecondPointerPosition( event ) {

			const pointer = ( event.pointerId === _pointers[ 0 ].pointerId ) ? _pointers[ 1 ] : _pointers[ 0 ];

			return _pointerPositions[ pointer.pointerId ];

		}

		this.dispose = function () {

			scope.domElement.removeEventListener( 'contextmenu', contextmenu );

			scope.domElement.removeEventListener( 'pointerdown', onPointerDown );
			scope.domElement.removeEventListener( 'pointercancel', onPointerCancel );
			scope.domElement.removeEventListener( 'wheel', onMouseWheel );

			scope.domElement.removeEventListener( 'pointermove', onPointerMove );
			scope.domElement.removeEventListener( 'pointerup', onPointerUp );

			window.removeEventListener( 'keydown', keydown );
			window.removeEventListener( 'keyup', keyup );

		};

		this.domElement.addEventListener( 'contextmenu', contextmenu );

		this.domElement.addEventListener( 'pointerdown', onPointerDown );
		this.domElement.addEventListener( 'pointercancel', onPointerCancel );
		this.domElement.addEventListener( 'wheel', onMouseWheel, { passive: false } );


		window.addEventListener( 'keydown', keydown );
		window.addEventListener( 'keyup', keyup );

		this.handleResize();

		// force an update at start
		this.update();

	}

}

class AsciiEffect {

	constructor( renderer, charSet = ' .:-=+*#%@', options = {} ) {

		// ' .,:;=|iI+hHOE#`$';
		// darker bolder character set from https://github.com/saw/Canvas-ASCII-Art/
		// ' .\'`^",:;Il!i~+_-?][}{1)(|/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$'.split('');

		// Some ASCII settings

		const bResolution = ! options[ 'resolution' ] ? 0.15 : options[ 'resolution' ]; // Higher for more details
		const iScale = ! options[ 'scale' ] ? 1 : options[ 'scale' ];
		const bColor = ! options[ 'color' ] ? false : options[ 'color' ]; // nice but slows down rendering!
		const bAlpha = ! options[ 'alpha' ] ? false : options[ 'alpha' ]; // Transparency
		const bBlock = ! options[ 'block' ] ? false : options[ 'block' ]; // blocked characters. like good O dos
		const bInvert = ! options[ 'invert' ] ? false : options[ 'invert' ]; // black is white, white is black

		const strResolution = 'low';

		let width, height;

		const domElement = document.createElement( 'div' );
		domElement.style.cursor = 'default';

		const oAscii = document.createElement( 'table' );
		domElement.appendChild( oAscii );

		let iWidth, iHeight;
		let oImg;

		this.setSize = function ( w, h ) {

			width = w;
			height = h;

			renderer.setSize( w, h );

			initAsciiSize();

		};


		this.render = function ( scene, camera ) {

			renderer.render( scene, camera );
			asciifyImage( oAscii );

		};

		this.domElement = domElement;


		// Throw in ascii library from https://github.com/hassadee/jsascii/blob/master/jsascii.js (MIT License)

		function initAsciiSize() {

			iWidth = Math.round( width * fResolution );
			iHeight = Math.round( height * fResolution );

			oCanvas.width = iWidth;
			oCanvas.height = iHeight;
			// oCanvas.style.display = "none";
			// oCanvas.style.width = iWidth;
			// oCanvas.style.height = iHeight;

			oImg = renderer.domElement;

			if ( oImg.style.backgroundColor ) {

				oAscii.rows[ 0 ].cells[ 0 ].style.backgroundColor = oImg.style.backgroundColor;
				oAscii.rows[ 0 ].cells[ 0 ].style.color = oImg.style.color;

			}

			oAscii.cellSpacing = 0;
			oAscii.cellPadding = 0;

			const oStyle = oAscii.style;
			oStyle.display = 'inline';
			oStyle.width = Math.round( iWidth / fResolution * iScale ) + 'px';
			oStyle.height = Math.round( iHeight / fResolution * iScale ) + 'px';
			oStyle.whiteSpace = 'pre';
			oStyle.margin = '0px';
			oStyle.padding = '0px';
			oStyle.letterSpacing = fLetterSpacing + 'px';
			oStyle.fontFamily = strFont;
			oStyle.fontSize = fFontSize + 'px';
			oStyle.lineHeight = fLineHeight + 'px';
			oStyle.textAlign = 'left';
			oStyle.textDecoration = 'none';

		}


		const aDefaultCharList = ( ' .,:;i1tfLCG08@' ).split( '' );
		const aDefaultColorCharList = ( ' CGO08@' ).split( '' );
		const strFont = 'courier new, monospace';

		const oCanvasImg = renderer.domElement;

		const oCanvas = document.createElement( 'canvas' );
		if ( ! oCanvas.getContext ) {

			return;

		}

		const oCtx = oCanvas.getContext( '2d' );
		if ( ! oCtx.getImageData ) {

			return;

		}

		let aCharList = ( bColor ? aDefaultColorCharList : aDefaultCharList );

		if ( charSet ) aCharList = charSet;

		let fResolution = 0.5;

		switch ( strResolution ) {

			case 'low' : 	fResolution = 0.25; break;
			case 'medium' : fResolution = 0.5; break;
			case 'high' : 	fResolution = 1; break;

		}

		if ( bResolution ) fResolution = bResolution;

		// Setup dom

		const fFontSize = ( 2 / fResolution ) * iScale;
		const fLineHeight = ( 2 / fResolution ) * iScale;

		// adjust letter-spacing for all combinations of scale and resolution to get it to fit the image width.

		let fLetterSpacing = 0;

		if ( strResolution == 'low' ) {

			switch ( iScale ) {

				case 1 : fLetterSpacing = - 1; break;
				case 2 :
				case 3 : fLetterSpacing = - 2.1; break;
				case 4 : fLetterSpacing = - 3.1; break;
				case 5 : fLetterSpacing = - 4.15; break;

			}

		}

		if ( strResolution == 'medium' ) {

			switch ( iScale ) {

				case 1 : fLetterSpacing = 0; break;
				case 2 : fLetterSpacing = - 1; break;
				case 3 : fLetterSpacing = - 1.04; break;
				case 4 :
				case 5 : fLetterSpacing = - 2.1; break;

			}

		}

		if ( strResolution == 'high' ) {

			switch ( iScale ) {

				case 1 :
				case 2 : fLetterSpacing = 0; break;
				case 3 :
				case 4 :
				case 5 : fLetterSpacing = - 1; break;

			}

		}


		// can't get a span or div to flow like an img element, but a table works?


		// convert img element to ascii

		function asciifyImage( oAscii ) {

			oCtx.clearRect( 0, 0, iWidth, iHeight );
			oCtx.drawImage( oCanvasImg, 0, 0, iWidth, iHeight );
			const oImgData = oCtx.getImageData( 0, 0, iWidth, iHeight ).data;

			// Coloring loop starts now
			let strChars = '';

			// console.time('rendering');

			for ( let y = 0; y < iHeight; y += 2 ) {

				for ( let x = 0; x < iWidth; x ++ ) {

					const iOffset = ( y * iWidth + x ) * 4;

					const iRed = oImgData[ iOffset ];
					const iGreen = oImgData[ iOffset + 1 ];
					const iBlue = oImgData[ iOffset + 2 ];
					const iAlpha = oImgData[ iOffset + 3 ];
					let iCharIdx;

					let fBrightness;

					fBrightness = ( 0.3 * iRed + 0.59 * iGreen + 0.11 * iBlue ) / 255;
					// fBrightness = (0.3*iRed + 0.5*iGreen + 0.3*iBlue) / 255;

					if ( iAlpha == 0 ) {

						// should calculate alpha instead, but quick hack :)
						//fBrightness *= (iAlpha / 255);
						fBrightness = 1;

					}

					iCharIdx = Math.floor( ( 1 - fBrightness ) * ( aCharList.length - 1 ) );

					if ( bInvert ) {

						iCharIdx = aCharList.length - iCharIdx - 1;

					}

					// good for debugging
					//fBrightness = Math.floor(fBrightness * 10);
					//strThisChar = fBrightness;

					let strThisChar = aCharList[ iCharIdx ];

					if ( strThisChar === undefined || strThisChar == ' ' )
						strThisChar = '&nbsp;';

					if ( bColor ) {

						strChars += '<span style=\''
							+ 'color:rgb(' + iRed + ',' + iGreen + ',' + iBlue + ');'
							+ ( bBlock ? 'background-color:rgb(' + iRed + ',' + iGreen + ',' + iBlue + ');' : '' )
							+ ( bAlpha ? 'opacity:' + ( iAlpha / 255 ) + ';' : '' )
							+ '\'>' + strThisChar + '</span>';

					} else {

						strChars += strThisChar;

					}

				}

				strChars += '<br/>';

			}

			oAscii.innerHTML = '<tr><td>' + strChars + '</td></tr>';

			// console.timeEnd('rendering');

			// return oAscii;

		}

	}

}

let camera, controls, scene, renderer, effect;

let sphere, plane;

const start = Date.now();

const init = () => {
	camera = new THREE.PerspectiveCamera( 70, window.innerWidth / window.innerHeight, 1, 1000 );
	camera.position.y = 50
	camera.position.z = 800

	scene = new THREE.Scene()

	const pointLight1 = new THREE.PointLight( 0xffffff )
	pointLight1.position.set(500,500,500)
	scene.add(pointLight1)

	const pointLight2 = new THREE.PointLight( 0xffffff, 0.25 )
	pointLight2.position.set(-500,-500,-500)
	scene.add(pointLight2)

	sphere = new THREE.Mesh( new THREE.SphereGeometry( 200, 20, 10 ), new THREE.MeshPhongMaterial({ flatShading: true }) )
	scene.add(sphere)

	plane = new THREE.Mesh( new THREE.PlaneGeometry(400, 400), new THREE.MeshBasicMaterial({ color: 0xe0e0e0 }) )
	plane.position.y = -200
	plane.rotation.x = -Math.PI / 2
	scene.add(plane)

	renderer = new THREE.WebGLRenderer( { alpha: true } )
	renderer.setSize( window.innerWidth, window.innerHeight )
	renderer.setClearColor(0xf0f0f0);

	effect = new AsciiEffect( renderer, " .:-+*=%@#")
	effect.setSize( window.innerWidth, window.innerHeight )
	effect.domElement.style.color = "black";
	effect.domElement.style.fontWeight = "500"

	document.getElementById("canvas").appendChild(effect.domElement)

	window.addEventListener( 'resize', onWindowResize );
}

function onWindowResize() {

	camera.aspect = window.innerWidth / window.innerHeight;
	camera.updateProjectionMatrix();

	renderer.setSize( window.innerWidth, window.innerHeight );
	effect.setSize( window.innerWidth, window.innerHeight );

}

function animate() {

	requestAnimationFrame( animate );

	render();

}

function render() {

	const timer = Date.now() - start;

	sphere.position.y = Math.abs( Math.sin( timer * 0.002 ) ) * 150;
	sphere.rotation.x = timer * 0.0003;
	sphere.rotation.z = timer * 0.0002;

	effect.render( scene, camera );

}

init()
animate()
        }
    }
</script>