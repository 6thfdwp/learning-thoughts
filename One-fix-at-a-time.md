[Fix varying column height in Bootstrap](https://medium.com/wdstack/varying-column-heights-in-bootstrap-4e8dd5338643)
--
>  The Bootstrap height issue occurs because the columns (col-*-*) use float:left. When a column is “floated” it’s taken out of the normal flow of the document. It is shifted to the left or right until it touches the edge of its containing box. So, when you have uneven column heights, the correct behavior is to stack them to the closest side.

1. Make column equal height

  could set fixed height if it works. Otherwise `flex` is best option if the height is unknown

```CSS
.row.display-flex {
  display: flex;
  flex-wrap: wrap;
}
.row.display-flex > [class*='col-'] {
  display: flex;
  flex-direction: column;
}
```

2. `clearfix` force column to wrap every X columns
> “clearfix” which basically removes the float from the rightmost column, so the next adjacent column can wrap properly to far left of the next row.

Official Bootstrap [responsive reset](http://getbootstrap.com/css/#grid-responsive-resets) already support this

```css
<div class='row'>
  <div class="col-md-4 col-xs-6"> 1 </div>
  <div class="col-md-4 col-xs-6"> 2 </div>
  <div class="clearfix visible-xs">
      /* <!-- clearfix xs cols every 2 --> */
  </div>
  <div class="col-md-4 col-xs-6"> 3 </div>
  <div class="clearfix visible-md">
      /* <!-- clearfix lg cols every 3 --> */
  </div>
</div>
```

If we want CSS only solution, we can use `nth-child`


[Overlay position](https://tympanus.net/codrops/2013/11/07/css-overlay-techniques/)
--
```css
<body>
  <div class='overlay'/>
  <div class='modal'>Modal content</div>
  <div class='content'>Long content</div>
</body>

html, body {
  /**
  full height relative to viewport, will increase for long content,
  so overlay always cover the whole as scrolling down
  **/
  min-height: 100%;
}
body {
  color: #fff;
  padding: 30px;
  position: relative;
}
.overlay {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  background-color: rgba(0,0,0,5);
  z-index: 10;
}
.modal {
  height: 300px; width: 300px;
  position: fixed;
  top: 50%; left: 50%;
  z-index: 11;
}
```
Need to have ancestor-descendant with relative-absolute relationship. Also the `body` need to set height (otherwise determined by its content), we may not have full-screen like overlay if content is too short or too long. This technique is useful for image overlay.

If we need full screen overlay as to viewport, use `position: fixed` is another way.
```CSS
body {
  color: #fff;
  padding: 30px;
}
.overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 10;
}
```
`fixed` is always relative to the initial containing block, no matter where you put it. the containing block is normally the viewport (browser window).
