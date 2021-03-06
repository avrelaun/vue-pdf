<template>
	<div style="position: relative"><canvas style="width: 100%; display: block"></canvas><div class="annotationLayer"></div></div>
</template>

<style>

/* see https://github.com/mozilla/pdf.js/blob/55a853b6678cf3d05681ffbb521e5228e607b5d2/test/annotation_layer_test.css */

.annotationLayer {
	position: absolute;
	left: 0;
	top: 0;
	right: 0;
	bottom: 0;
}

.annotationLayer section {
  position: absolute;
}

.annotationLayer .linkAnnotation > a {
  position: absolute;
  font-size: 1em;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.annotationLayer .linkAnnotation > a /* -ms-a */  {
  background: url("data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7") 0 0 repeat;
}

.annotationLayer .linkAnnotation > a:hover {
  opacity: 0.2;
  background: #ff0;
  box-shadow: 0px 2px 10px #ff0;
}

.annotationLayer .textAnnotation img {
  position: absolute;
  cursor: pointer;
}

.annotationLayer .textWidgetAnnotation input,
.annotationLayer .textWidgetAnnotation textarea,
.annotationLayer .choiceWidgetAnnotation select,
.annotationLayer .buttonWidgetAnnotation.checkBox input,
.annotationLayer .buttonWidgetAnnotation.radioButton input {
  background-color: rgba(0, 54, 255, 0.13);
  border: 1px solid transparent;
  box-sizing: border-box;
  font-size: 9px;
  height: 100%;
  padding: 0 3px;
  vertical-align: top;
  width: 100%;
}

.annotationLayer .textWidgetAnnotation textarea {
  font: message-box;
  font-size: 9px;
  resize: none;
}

.annotationLayer .textWidgetAnnotation input[disabled],
.annotationLayer .textWidgetAnnotation textarea[disabled],
.annotationLayer .choiceWidgetAnnotation select[disabled],
.annotationLayer .buttonWidgetAnnotation.checkBox input[disabled],
.annotationLayer .buttonWidgetAnnotation.radioButton input[disabled] {
  background: none;
  border: 1px solid transparent;
  cursor: not-allowed;
}

.annotationLayer .textWidgetAnnotation input:hover,
.annotationLayer .textWidgetAnnotation textarea:hover,
.annotationLayer .choiceWidgetAnnotation select:hover,
.annotationLayer .buttonWidgetAnnotation.checkBox input:hover,
.annotationLayer .buttonWidgetAnnotation.radioButton input:hover {
  border: 1px solid #000;
}

.annotationLayer .textWidgetAnnotation input:focus,
.annotationLayer .textWidgetAnnotation textarea:focus,
.annotationLayer .choiceWidgetAnnotation select:focus {
  background: none;
  border: 1px solid transparent;
}

.annotationLayer .textWidgetAnnotation input.comb {
  font-family: monospace;
  padding-left: 2px;
  padding-right: 0;
}

.annotationLayer .textWidgetAnnotation input.comb:focus {
  /*
   * Letter spacing is placed on the right side of each character. Hence, the
   * letter spacing of the last character may be placed outside the visible
   * area, causing horizontal scrolling. We avoid this by extending the width
   * when the element has focus and revert this when it loses focus.
   */
  width: 115%;
}

.annotationLayer .buttonWidgetAnnotation.checkBox input,
.annotationLayer .buttonWidgetAnnotation.radioButton input {
  -webkit-appearance: none;
  -moz-appearance: none;
  -ms-appearance: none;
  appearance: none;
}

.annotationLayer .popupWrapper {
  position: absolute;
  width: 20em;
}

.annotationLayer .popup {
  position: absolute;
  z-index: 200;
  max-width: 20em;
  background-color: #FFFF99;
  box-shadow: 0px 2px 5px #333;
  border-radius: 2px;
  padding: 0.6em;
  margin-left: 5px;
  cursor: pointer;
  word-wrap: break-word;
}

.annotationLayer .popup h1 {
  font-size: 1em;
  border-bottom: 1px solid #000000;
  padding-bottom: 0.2em;
}

.annotationLayer .popup p {
  padding-top: 0.2em;
}

.annotationLayer .highlightAnnotation,
.annotationLayer .underlineAnnotation,
.annotationLayer .squigglyAnnotation,
.annotationLayer .strikeoutAnnotation,
.annotationLayer .lineAnnotation svg line,
.annotationLayer .fileAttachmentAnnotation {
  cursor: pointer;
}
</style>

<script>
"use strict";

const PDFJS = require('pdfjs-dist/webpack.js');

function PDFJSWrapper(PDFJS, canvasElt, annotationLayerElt, emitEvent) {
	
	var pdfDoc = null;
	var pdfPage = null;
	var pdfRender = null;

	function clearCanvas() {
		
		canvasElt.getContext('2d').clearRect(0, 0, canvasElt.width, canvasElt.height);
	}
	
	function clearAnnotations() {
		
		while ( annotationLayerElt.firstChild )
			annotationLayerElt.removeChild(annotationLayerElt.firstChild);
	}
	
	this.destroy = function() {
		
		if ( pdfDoc === null )
			return;
		pdfDoc.destroy();
		pdfDoc = null;
	}
	
	this.getResolutionScale = function() {
		
		return canvasElt.offsetWidth / canvasElt.width;
	}

	this.printPage = function(dpi, pageNumberOnly) {

		if ( pdfPage === null )
			return;
		
		// 1in == 72pt
		// 1in == 96px
		var PRINT_RESOLUTION = dpi === undefined ? 150 : dpi;
		var PRINT_UNITS = PRINT_RESOLUTION / 72.0;
		var CSS_UNITS = 96.0 / 72.0;
		
		var iframeElt = document.createElement('iframe');

		function removeIframe() {

			iframeElt.parentNode.removeChild(iframeElt);
		}
		
		new Promise(function(resolve, reject) {

			iframeElt.frameBorder = '0';
			iframeElt.scrolling = 'no';
			iframeElt.width = '0px;' 
			iframeElt.height = '0px;'
			iframeElt.style.cssText = 'position: absolute; top: 0; left: 0';

			iframeElt.onload = function() {
				
				resolve(this.contentWindow);
			}
			
			window.document.body.appendChild(iframeElt);
		})
		.then(function(win) {
			
			win.document.title = '';

			return pdfDoc.getPage(1)
			.then(function(page) {
				
				var viewport = page.getViewport(1);
				win.document.head.appendChild(win.document.createElement('style')).textContent = 
					'@supports ((size:A4) and (size:1pt 1pt)) {' +
						'@page { margin: 1pt; size: ' + ((viewport.width * PRINT_UNITS) / CSS_UNITS) + 'pt ' + ((viewport.height * PRINT_UNITS) / CSS_UNITS) + 'pt; }' +
					'}' +

					'@media print {' +
						'body { margin: 0 }' +
						'canvas { page-break-before: avoid; page-break-after: always; page-break-inside: avoid }' +
					'}'+

					'@media screen {' +
						'body { margin: 0 }' +
					'}'+

					''
				return win;
			})
		})
		.then(function(win) {
			
			var allPages = [];
			
			for ( var pageNumber = 1; pageNumber <= pdfDoc.numPages; ++pageNumber ) {
				
				if ( pageNumberOnly !== undefined && pageNumberOnly.indexOf(pageNumber) === -1 )
					continue;
				
				allPages.push(
					pdfDoc.getPage(pageNumber)
					.then(function(page) {

						var viewport = page.getViewport(1);
					
						var printCanvasElt = win.document.body.appendChild(win.document.createElement('canvas'));
						printCanvasElt.width = (viewport.width * PRINT_UNITS);
						printCanvasElt.height = (viewport.height * PRINT_UNITS);

						return page.render({
							canvasContext: printCanvasElt.getContext('2d'),
							transform: [ // Additional transform, applied just before viewport transform.
								PRINT_UNITS, 0, 0,
								PRINT_UNITS, 0, 0
							],
							viewport: viewport,
							intent: 'print'
						}).promise;
					})
				);
			}
			
			Promise.all(allPages)
			.then(function() {
				
				win.focus(); // Required for IE
				win.print();
				removeIframe();
			})
			.catch(function(err) {
			
				removeIframe();
				emitEvent('error', err);
			})
		})
	}
	
	this.renderPage = function(rotate) {
		
		if ( pdfRender !== null )
			return pdfRender.cancel();

		if ( pdfPage === null )
			return;

		if ( rotate === undefined )
			rotate = 0;

		var viewport = pdfPage.getViewport(canvasElt.offsetWidth / pdfPage.getViewport(1).width, rotate);

		emitEvent('pageSize', viewport.width, viewport.height);
		
		canvasElt.width = viewport.width;
		canvasElt.height = viewport.height;

		pdfRender = pdfPage.render({
			canvasContext: canvasElt.getContext('2d'),
			viewport: viewport
		});
		
		annotationLayerElt.style.visibility = 'hidden';
		clearAnnotations();
		
		pdfPage.getAnnotations()
		.then(function(annotations) {

			PDFJS.AnnotationLayer.render({
				viewport: viewport.clone({ dontFlip: true }),
				div: annotationLayerElt,
				annotations: annotations,
				page: pdfPage,
				//linkService: new LinkServiceMock(),
				renderInteractiveForms: false,
			});
		});

		pdfRender
		.then(function() {
			
			annotationLayerElt.style.visibility = '';
			pdfRender = null;
		})
		.catch(function(err) {

			pdfRender = null;
			if ( err === 'cancelled' )
				return this.renderPage(rotate);
			emitEvent('error', err);
		}.bind(this))
	}		

	this.loadPage = function(pageNumber, rotate) {
		
		pdfPage = null;
		
		if ( pdfDoc === null )
			return;
		
		pdfDoc.getPage(pageNumber)
		.then(function(page) {

			pdfPage = page;
			this.renderPage(rotate);
			emitEvent('pageLoaded', page.pageNumber);
		}.bind(this))
		.catch(function(err) {
			
			clearCanvas();
			clearAnnotations();
			emitEvent('error', err);
		});
	}

	this.loadDocument = function(src) {
		
		pdfDoc = null;
		pdfPage = null;
		
		emitEvent('numPages', undefined);

		if ( !src ) {
			
			canvasElt.removeAttribute('width');
			canvasElt.removeAttribute('height');
			clearAnnotations();
			return;
		}
		
		var loadingTask = PDFJS.getDocument(src);
		
		loadingTask.onPassword = function(updatePassword, reason) {
			
			var reasonStr;
			switch (reason) {
				case PDFJS.PasswordResponses.NEED_PASSWORD:
					reasonStr = 'NEED_PASSWORD';
					break;
				case PDFJS.PasswordResponses.INCORRECT_PASSWORD:
					reasonStr = 'INCORRECT_PASSWORD';
					break;
			}
			emitEvent('password', updatePassword, reasonStr);
		};
		
		loadingTask.onProgress = function(status) {
			
			var ratio = status.loaded / status.total;
			emitEvent('progress', Math.min(ratio, 1));
		}
		
		loadingTask
		.then(function(pdf) {
			
			pdfDoc = pdf;
			emitEvent('numPages', pdf.numPages);
			emitEvent('loaded');
		})
		.catch(function(err) {
			
			clearCanvas();
			clearAnnotations();
			emitEvent('error', err);
		})
		.then(function() {
			
			loadingTask._worker = null;
		})
		
	}
	
	PDFJS.CustomStyle.setProp('transform-origin', annotationLayerElt, '0 0');
}

module.exports = {
	props: {
		src: {
			type: [String, Object],
			default: '',
		},
		page: {
			type: Number,
			default: 1,
		},
		rotate: {
			type: Number,
			default: 0,
		},
		password: {
			type: Function,
			default: null,
		},
	},
	watch: {
		src: function() {
			
			this.pdf.loadDocument(this.src);
		},
		page: function() {
			
			this.pdf.loadPage(this.page, this.rotate);
		},
		rotate: function() {
			
			this.pdf.renderPage(this.rotate);
		},
	},
	methods: {
		resize: function(size) {
	
			var canvasElt = this.$el.childNodes[0];
			var annotationLayerElt = this.$el.childNodes[1];

			// on IE10- canvas height must be set
			canvasElt.style.height = canvasElt.offsetWidth * (canvasElt.height / canvasElt.width) + 'px';

			// update the page when the resolution is too poor
			var resolutionScale = this.pdf.getResolutionScale();
			
			if ( resolutionScale < 0.85 || resolutionScale > 1.15 )
				this.pdf.renderPage(this.rotate);

			PDFJS.CustomStyle.setProp('transform', annotationLayerElt, 'scale('+resolutionScale+')');
		},
		print: function(dpi, pageList) {

			this.pdf.printPage(dpi, pageList);
		}
	},
	mounted: function() {
		
		var canvasElt = this.$el.childNodes[0];
		var annotationLayerElt = this.$el.childNodes[1];
		
		this.pdf = new PDFJSWrapper(PDFJS, canvasElt, annotationLayerElt, this.$emit.bind(this));
		
		this.$on('loaded', function() {
			
			this.pdf.loadPage(this.page, this.rotate);
		});
		
		this.$on('pageSize', function(width, height) {
			
			canvasElt.style.height = canvasElt.offsetWidth * (height / width) + 'px';
		});
		
		this.pdf.loadDocument(this.src);
	},
	destroyed: function() {
		
		this.pdf.destroy();
	}
}
</script>
