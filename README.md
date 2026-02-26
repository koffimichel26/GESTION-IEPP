# GESTION-IEPP
Logiciel de gestion de circonscription scolaire C.I

<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>IEPP Manager V3</title>

<!-- jsPDF pour export PDF -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js"></script>

<style>
body{font-family:Arial;margin:0;background:#f5f5f5;}
header{background:#FFC107;padding:15px;text-align:center;font-weight:bold;font-size:20px;color:#000;}
.container{padding:15px;}
.card{background:white;padding:15px;margin-bottom:15px;border-radius:10px;box-shadow:0 3px 8px rgba(0,0,0,0.1);}
h3{color:#FFC107;margin-top:0;}
input,textarea,select{width:100%;padding:8px;margin-bottom:10px;border-radius:6px;border:1px solid #ddd;}
button{width:100%;padding:10px;background:#FFC107;border:none;border-radius:6px;font-weight:bold;cursor:pointer;}
button:hover{background:#e0a800;}
ul{list-style:none;padding:0;}
li{padding:8px;border-bottom:1px solid #eee;font-size:14px;}
.dashboard{display:flex;justify-content:space-between;background:#FFF3CD;padding:10px;border-radius:8px;margin-bottom:15px;font-weight:bold;}
.stat-box{background:#fff8e1;padding:8px;border-radius:6px;margin-bottom:5px;}
.teacher-box{background:#fff3cd;padding:6px;margin:4px;border-radius:4px;}
</style>
</head>
<body>

<header>Inspection Enseignement Préscolaire et Primaire</header>

<div class="container">

<!-- Tableau de bord -->
<div class="dashboard">
<div>Ecoles : <span id="totalSchools">0</span></div>
<div>Visites : <span id="totalVisits">0</span></div>
<div>Enseignants : <span id="totalTeachers">0</span></div>
</div>

<!-- Ajouter une école -->
<div class="card">
<h3>🏫 Ajouter une école</h3>
<input type="text" id="schoolName" placeholder="Nom de l'école">
<input type="text" id="schoolDirector" placeholder="Nom du directeur">
<input type="number" id="schoolStudents" placeholder="Nombre total d'élèves">
<input type="number" id="schoolGirls" placeholder="Nombre de filles">
<input type="number" id="schoolBoys" placeholder="Nombre de garçons">
<input type="number" id="schoolCEPE" placeholder="Nombre de candidats CEPE">
<button onclick="addSchool()">Ajouter l'école</button>
</div>

<!-- Liste des écoles -->
<div class="card">
<h3>📋 Liste des écoles</h3>
<ul id="schoolList"></ul>
</div>

<!-- Ajouter enseignants -->
<div class="card">
<h3>👩‍🏫 Ajouter un enseignant</h3>
<select id="teacherSchool"></select>
<input type="text" id="teacherName" placeholder="Nom enseignant">
<input type="text" id="teacherContact" placeholder="Contact enseignant">
<button onclick="addTeacher()">Ajouter l'enseignant</button>
</div>

<!-- Visites pédagogiques -->
<div class="card">
<h3>📝 Nouvelle visite pédagogique</h3>
<select id="visitSchool"></select>
<textarea id="visitNote" placeholder="Observation et recommandations"></textarea>
<button onclick="saveVisit()">Enregistrer la visite</button>
</div>

<!-- Statistiques par école -->
<div class="card">
<h3>📊 Statistiques par école</h3>
<div id="stats"></div>
</div>

<!-- Export PDF -->
<div class="card">
<h3>📄 Exporter Rapport PDF</h3>
<button onclick="exportPDF()">Télécharger PDF</button>
</div>

</div>

<script type="module">

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore, collection, addDoc, getDocs } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

// 🔥 Remplace par ta config Firebase
const firebaseConfig = {
  apiKey: "TON_API_KEY",
  authDomain: "TON_AUTH_DOMAIN",
  projectId: "TON_PROJECT_ID",
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

// Données locales
let schools = [];
let visits = [];

// Charger les visites depuis Firebase
async function loadVisits(){
  const querySnapshot = await getDocs(collection(db, "visits"));
  querySnapshot.forEach((doc)=>{
    visits.push(doc.data());
    if(!schools.find(s=>s.name===doc.data().school)) schools.push({name:doc.data().school,director:"",students:0,girls:0,boys:0,cepe:0,teachersList:[]});
  });
  updateUI();
}

// Mettre à jour tableau de bord et listes
function updateUI(){
  document.getElementById("totalSchools").innerText = schools.length;
  document.getElementById("totalVisits").innerText = visits.length;
  let totalTeachers = schools.reduce((acc,s)=>acc + (s.teachersList? s.teachersList.length:0),0);
  document.getElementById("totalTeachers").innerText = totalTeachers;

  let visitSelect = document.getElementById("visitSchool");
  let teacherSelect = document.getElementById("teacherSchool");
  visitSelect.innerHTML="";
  teacherSelect.innerHTML="";
  schools.forEach(s=>{
    let option1=document.createElement("option");
    option1.value=s.name;
    option1.textContent=s.name;
    visitSelect.appendChild(option1);

    let option2=document.createElement("option");
    option2.value=s.name;
    option2.textContent=s.name;
    teacherSelect.appendChild(option2);
  });

  let list = document.getElementById("schoolList");
  list.innerHTML="";
  schools.forEach(s=>{
    let li=document.createElement("li");
    li.innerHTML=`<strong>${s.name}</strong> - Directeur: ${s.director} - Élèves: ${s.students} (Filles: ${s.girls}, Garçons: ${s.boys}) - CEPE: ${s.cepe} - Enseignants: ${s.teachersList.length}<br>`;
    if(s.teachersList.length>0){
      s.teachersList.forEach(t=>{
        li.innerHTML+=`<div class="teacher-box">${t.name} - ${t.contact}</div>`;
      });
    }
    list.appendChild(li);
  });

  let statsDiv = document.getElementById("stats");
  statsDiv.innerHTML="";
  schools.forEach(s=>{
    let count = visits.filter(v=>v.school===s.name).length;
    statsDiv.innerHTML += `<div class="stat-box">${s.name} : ${count} visite(s)</div>`;
  });
}

// Ajouter école
window.addSchool = function(){
  let name = document.getElementById("schoolName").value.trim();
  let director = document.getElementById("schoolDirector").value.trim();
  let students = parseInt(document.getElementById("schoolStudents").value)||0;
  let girls = parseInt(document.getElementById("schoolGirls").value)||0;
  let boys = parseInt(document.getElementById("schoolBoys").value)||0;
  let cepe = parseInt(document.getElementById("schoolCEPE").value)||0;
  if(name){
    schools.push({name,director,students,girls,boys,cepe,teachersList:[]});
    updateUI();
    // Reset
    document.getElementById("schoolName").value="";
    document.getElementById("schoolDirector").value="";
    document.getElementById("schoolStudents").value="";
    document.getElementById("schoolGirls").value="";
    document.getElementById("schoolBoys").value="";
    document.getElementById("schoolCEPE").value="";
  }
}

// Ajouter enseignant
window.addTeacher = function(){
  let schoolName=document.getElementById("teacherSchool").value;
  let name=document.getElementById("teacherName").value.trim();
  let contact=document.getElementById("teacherContact").value.trim();
  if(name && schoolName){
    let school = schools.find(s=>s.name===schoolName);
    if(school){
      school.teachersList.push({name,contact});
      updateUI();
      document.getElementById("teacherName").value="";
      document.getElementById("teacherContact").value="";
    }
  }
}

// Enregistrer visite
window.saveVisit = async function(){
  let school=document.getElementById("visitSchool").value;
  let note=document.getElementById("visitNote").value.trim();
  let date=new Date().toLocaleDateString();
  if(note){
    let visit={school,note,date};
    visits.push(visit);
    await addDoc(collection(db,"visits"),visit);
    document.getElementById("visitNote").value="";
    updateUI();
  }
}

// Export PDF
window.exportPDF = function(){
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();
  doc.text("Rapport IEPP",10,10);
  let y=20;
  schools.forEach(s=>{
    doc.text(`${s.name} - Directeur: ${s.director} - Élèves: ${s.students} (Filles: ${s.girls}, Garçons: ${s.boys}) - CEPE: ${s.cepe}`,10,y);
    y+=7;
    s.teachersList.forEach(t=>{
      doc.text(`Enseignant: ${t.name} - ${t.contact}`,10,y);
      y+=5;
    });
    let visitCount = visits.filter(v=>v.school===s.name).length;
    doc.text(`Visites: ${visitCount}`,10,y);
    y+=10;
  });
  doc.save("rapport_iepp_v3.pdf");
}

loadVisits();

</script>
</body>
</html>
