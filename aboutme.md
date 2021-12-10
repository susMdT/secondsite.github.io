---
layout: page
title: About me
---
<head>
<style> /*center text, make 3 columns of equal width, remove the white border this theme has by default*/
th {text-align: center; border-bottom: 0px;}
td {text-align: center; border-bottom: 0px;}
</style>
<script>
function show() {
  var x = document.getElementById("carouselExampleIndicators");
  if (x.style.display === "none") {
    x.style.display = "block";
  } else {
    x.style.display = "none";
  }
}
</script>
</head>

My name is Dylan Tran. I'm attending Cal Poly Pomona and am interested in Offensive Security.

- Active Directory is pretty cool
- Linux is neat
- I also go by the handle Nigerald
- Self proclaimed HackTheBox addict

<div onClick="show()" id="hovere"><h2>Achievements</h2></div>


<div id="carouselExampleIndicators" class="carousel slide" data-ride="carousel" style="display: none">
  <ol class="carousel-indicators" style="top: -10%">
    <li data-target="#carouselExampleIndicators" data-slide-to="0" class="active"></li>
    <li data-target="#carouselExampleIndicators" data-slide-to="1" class=""></li>
    <li data-target="#carouselExampleIndicators" data-slide-to="2" class=""></li>
  </ol>
  <div class="carousel-inner">
    <div class="carousel-item active">
      <div class="d-block w-100" style="height: 500px">
        <img class="d-block w-100" alt="First slide" src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/CPTC.png?raw=true"/>
        <br/>
        <h5 style="text-align: center">CPTC Western Regional Championships, 2021</h5>
      </div>
    </div>
    <div class="carousel-item">
      <div class="d-block w-100" style="height: 500px">
        <img class="d-block w-100" alt="Second slide" src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/CCDC.png?raw=true"/>
        <br/>
        <h5 style="text-align: center">6th CCDC First Invitationals, 2021</h5>
      </div>
    </div>
    <div class="carousel-item">
      <div class="d-block w-100" style="height: 500px">
        <img class="d-block w-100" alt="Third slide" src="https://github.com/susMdT/secondsite.github.io/blob/master/assets/img/Hivestorm.png?raw=true"/>
        <br/>
        <h5 style="text-align: center">5th Hivestorm, 2021</h5>
      </div>
    </div>
  </div>
  <a class="carousel-control-prev" href="#carouselExampleIndicators" style="top: -50%"role="button" data-slide="prev">
    <span class="carousel-control-prev-icon" aria-hidden="true"></span>
    <span class="sr-only">Previous</span>
  </a>
  <a class="carousel-control-next" href="#carouselExampleIndicators" role="button" style="top: -50%" data-slide="next">
    <span class="carousel-control-next-icon" aria-hidden="true"></span>
    <span class="sr-only">Next</span>
  </a>
</div>
