---
layout: default
title:  "NimES: a NES emulator in Nim, compiled to JavaScript"
---
# NimES: a NES emulator in Nim, compiled to JavaScript

<p>This is <a href="https://github.com/def-/nimes">NimES</a>, a NES emulator in <a href="http://nim-lang.org/">Nim</a>, compiled to JavaScript using emscripten (<a href="http://hookrace.net/blog/porting-nes-go-nim/">accompanying blog post</a>):</p>
<div style="text-align: center;"><canvas style="width: 256px; height: 240px;" class="emscripten" id="canvas" oncontextmenu="event.preventDefault()"></canvas></div>
<p>Games: <a href="?nes=smb.nes">Super Mario Bros</a>, <a href="?nes=smb3.nes">Super Mario Bros 3</a>, <a href="?nes=tetris.nes">Tetris</a>, <a href="?nes=pacman.nes">Pacman</a></p>
<table style="margin-left: auto; margin-right: auto;">
  <tr><th>Key</th><th>Action</th></tr>
  <tr><td>←↑↓→</td><td>←↑↓→</td></tr>
  <tr><td>Z/Y</td><td>A</td></tr>
  <tr><td>X</td><td>B</td></tr>
  <tr><td>Enter</td><td>Start</td></tr>
  <tr><td>Space</td><td>Select</td></tr>
  <tr><td>1-5</td><td>Zoom 1-5×</td></tr>
  <tr><td>R</td><td>Reset</td></tr>
  <tr><td>P</td><td>Pause</td></tr>
</table>
<script type='text/javascript'>
  var canvas = document.getElementById('canvas');
  canvas.setAttribute('width', window.innerWidth);
  canvas.setAttribute('height', window.innerHeight);

  var QueryString = function () {
    // This function is anonymous, is executed immediately and 
    // the return value is assigned to QueryString!
    var query_string = {};
    var query = window.location.search.substring(1);
    var vars = query.split("&");
    for (var i=0;i<vars.length;i++) {
      var pair = vars[i].split("=");
          // If first entry with this name
      if (typeof query_string[pair[0]] === "undefined") {
        query_string[pair[0]] = pair[1];
          // If second entry with this name
      } else if (typeof query_string[pair[0]] === "string") {
        var arr = [ query_string[pair[0]], pair[1] ];
        query_string[pair[0]] = arr;
          // If third or later entry with this name
      } else {
        query_string[pair[0]].push(pair[1]);
      }
    }
    return query_string;
  } ();

  var argument;
  if (QueryString.hasOwnProperty("nes")) {
    argument = QueryString.nes;
  } else {
    argument = "smb3.nes";
  }

  var Module;

  Module = {
    preRun: [],
    postRun: [],
    arguments: [argument],
    canvas: (function() {
      var canvas = document.getElementById('canvas');
      canvas.addEventListener("webglcontextlost", function(e) { alert('WebGL context lost. You will need to reload the page.'); e.preventDefault(); }, false);
      return canvas;
    })(),
    totalDependencies: 0
  };

  window.onerror = function(event) {};
</script>
<script async type="text/javascript" src="nimes.js"></script>
