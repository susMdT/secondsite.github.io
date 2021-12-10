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
  var x = document.getElementById("achievements");
  if (x.style.display === "none") {
    x.style.display = "block";
  } else {
    x.style.display = "none";
  }
}
</script>
</head>

My name is Dylan Tran. I'm attending Cal Poly Pomona and am interested in Offensive Security.

- I rock a great mustache
- I'm extremely loyal to my family

What else do you need?

### My Achivements

<div onClick="show()" id="hovere"><h2>Achievements</h2></div>

<div id="achievements" style="display:none">
  <div id="sideGrouper">
    <div id="indexFloat">
      <img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/CPTC_Logo.png?raw=true" width="100%" height="100%" unselectable="on" />
    </div>
    <div id="indexFloat">
      <table>
        <tr>
          <th colspan="3"><h1 style="font-size:20px">Collegiate Penetration Testing Competition</h1></th>    
        </tr>
        <tr>
          <td style="width: 40%">Western Regionals</td>
          <td style="width: 33%">1st Place</td>
          <td style="width: 33%">2021</td>
        </tr>
      </table>
    </div>
  </div>
  <div id="sideGrouper">
    <div id="indexFloat">
      <img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Hivestorm_Logo.png?raw=true" width="100%" height="100%" unselectable="on" />
    </div>
    <div id="indexFloat">
      <table>
        <tr>
          <th colspan="3"><h1 style="font-size:20px">Hivestorm</h1></th>    
        </tr>
        <tr>
          <td style="width: 50%">5th Place</td>
          <td style="width: 50%">2021</td>
        </tr>
      </table>
    </div>
  </div>
  <div id="sideGrouper">
    <div id="indexFloat">
      <img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/CCDC.jpg?raw=true" width="100%" height="100%" unselectable="on" />
    </div>
    <div id="indexFloat">
      <table>
        <tr>
          <th colspan="3"><h1 style="font-size:20px">Collegiate Cyber Defense Competition</h1></th>    
        </tr>
        <tr>
          <td style="width: 40%">Western Regional Invitationals</td>
          <td style="width: 33%">6th Place</td>
          <td style="width: 33%">2021</td>
        </tr>
      </table>
    </div>
  </div> 
</div>
