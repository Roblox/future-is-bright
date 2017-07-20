---
layout: default
title: Future Is Bright
---

<p id="demo"></p>

| Hack Week 2015 (Next Generation Voxels) | Hack Week 2016 (Future Is Bright) |
|:-:|:-:|
| [![](https://img.youtube.com/vi/z5TmqDtpwSM/0.jpg)](https://www.youtube.com/watch?v=z5TmqDtpwSM) | [![](https://img.youtube.com/vi/lrvOGqC9ZjQ/0.jpg)](https://www.youtube.com/watch?v=lrvOGqC9ZjQ) |

Content here
[Link here](https://github.com/Roblox/future-is-bright/releases/download/v1/future-is-bright-v1.zip)

<script>
// Set the date we're counting down to
var countDownDate = new Date("Jan 5, 2018 15:37:25").getTime();

// Update the count down every 1 second
var x = setInterval(function() {

  // Get todays date and time
  var now = new Date().getTime();

  // Find the distance between now an the count down date
  var distance = countDownDate - now;

  // Time calculations for days, hours, minutes and seconds
  var days = Math.floor(distance / (1000 * 60 * 60 * 24));
  var hours = Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
  var minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60));
  var seconds = Math.floor((distance % (1000 * 60)) / 1000);

  // Display the result in the element with id="demo"
  document.getElementById("demo").innerHTML = days + "d " + hours + "h "
  + minutes + "m " + seconds + "s ";

  // If the count down is finished, write some text 
  if (distance < 0) {
    clearInterval(x);
    document.getElementById("demo").innerHTML = "EXPIRED";
  }
}, 1000);
</script>
