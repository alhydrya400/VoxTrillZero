<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>VoxTrill Zero — المترجم العالمي</title>
<style>
body{font-family:Arial,Helvetica,sans-serif;direction:rtl;padding:18px;background:#f5f7fb;color:#111}
.container{max-width:820px;margin:0 auto}
h1{text-align:center}
textarea{width:100%;height:110px;padding:10px;font-size:16px;border-radius:8px;border:1px solid #ccc;box-sizing:border-box}
.row{display:flex;gap:10px;flex-wrap:wrap;margin-top:10px;align-items:center}
button{padding:10px 14px;border-radius:8px;border:none;background:#2b8cff;color:white;cursor:pointer}
#output,#pron{background:#fff;padding:12px;border-radius:8px;border:1px solid #d0d7e6;min-height:48px;margin-top:10px;word-break:break-word}
.small{font-size:13px;color:#555;margin-top:8px}
label{display:block;margin-top:10px}
select{padding:8px;border-radius:6px}
</style>
</head>
<body>
<div class="container">
  <h1>VoxTrill Zero — المترجم العالمي</h1>

  <textarea id="inputText" placeholder="اكتب أي نص بأي لغة هنا..."></textarea>

  <div class="row">
    <button id="playBtn">استمع إلى النطق</button>
    <div style="margin-left:auto">
      <label>اختيار صوت (Voice):</label>
      <select id="voiceSelect"></select>
    </div>
  </div>

  <div id="output" aria-live="polite"></div>
  <div id="pron" aria-live="polite" class="small"></div>

  <p class="small">ملاحظة: يعمل فقط على المتصفحات الحديثة (Chrome/Edge/Firefox). اختر صوت من القائمة إذا لم تسمع شيئًا.</p>
</div>

<script>
/* ================= جدول VoxTrill Zero 0..100 ================ */
const voxZero={
0:"za",1:"a",2:"i",3:"u",4:"e",5:"o",6:"m",7:"n",8:"s",9:"l",10:"r",
11:"ra",12:"ri",13:"ru",14:"re",15:"ro",16:"ma",17:"mi",18:"mu",19:"me",20:"zi",
21:"za",22:"zi",23:"zu",24:"ze",25:"zo",26:"ta",27:"ti",28:"tu",29:"te",30:"to",
31:"ka",32:"ki",33:"ku",34:"ke",35:"ko",36:"fa",37:"fi",38:"fu",39:"fe",40:"fo",
41:"ba",42:"bi",43:"bu",44:"be",45:"bo",46:"da",47:"di",48:"du",49:"de",50:"do",
51:"ga",52:"gi",53:"gu",54:"ge",55:"go",56:"ha",57:"hi",58:"hu",59:"he",60:"ho",
61:"ya",62:"yi",63:"yu",64:"ye",65:"yo",66:"wa",67:"wi",68:"wu",69:"we",70:"wo",
71:"sa",72:"si",73:"su",74:"se",75:"so",76:"na",77:"ni",78:"nu",79:"ne",80:"no",
81:"la",82:"li",83:"lu",84:"le",85:"lo",86:"pa",87:"pi",88:"pu",89:"pe",90:"po",
91:"ra",92:"ri",93:"ru",94:"re",95:"ro",96:"za",97:"zi",98:"zu",99:"ze",100:"zo"
};

/* ================= تحويل النصوص العالمية إلى أرقام ================= */
function textToNumbers(text){
  const nums=[];
  for(let ch of text){
    if(/\s/.test(ch)){nums.push(null);continue;}
    const code = ch.charCodeAt(0);
    if(code>=65 && code<=90){nums.push((code-65)%100 +1); continue;} // A-Z
    if(code>=97 && code<=122){nums.push((code-97)%100 +1); continue;} // a-z
    if(code>=0x0600 && code<=0x06FF){ nums.push((code-0x0600)%100 +1); continue; } // Arabic Unicode approx
    if(/[0-9]/.test(ch)){ nums.push(Number(ch)); continue; }
  }
  return nums;
}

/* ================= تحويل الأرقام إلى Vox ================= */
function numbersToVox(nums){
  const sylls=[];
  const displayNums=[];
  for(let n of nums){
    if(n===null){ displayNums.push("/"); sylls.push("/"); continue; }
    if(n in voxZero){ displayNums.push(String(n)); sylls.push(voxZero[n]); }
    else{ // fallback
      const digits = String(n).split("").map(d=>Number(d));
      const parts = digits.map(d=> voxZero[d] || d);
      displayNums.push(String(n));
      sylls.push(parts.join("-"));
    }
  }
  return {displayNums,sylls};
}

/* ================= واجهة المستخدم ================= */
const input=document.getElementById("inputText");
const out=document.getElementById("output");
const pron=document.getElementById("pron");
const playBtn=document.getElementById("playBtn");
const voiceSelect=document.getElementById("voiceSelect");

function refreshVoices(){
  const voices=speechSynthesis.getVoices();
  voiceSelect.innerHTML="";
  voices.forEach((v,idx)=>{
    const opt=document.createElement("option");
    opt.value=idx;
    opt.textContent=`${v.name} — ${v.lang}`;
    voiceSelect.appendChild(opt);
  });
}
speechSynthesis.onvoiceschanged=refreshVoices;
refreshVoices();

function render(){
  const text=input.value||"";
  const nums=textToNumbers(text);
  const mapped=numbersToVox(nums);
  out.innerText=mapped.displayNums.join(" ");
  pron.innerText=mapped.sylls.join(" · ");
  return mapped;
}

input.addEventListener("input", render);

async function playVox(){
  const mapped=render();
  const syllables=mapped.sylls.filter(s=>s!="/");
  if(syllables.length===0) return;
  const voices=speechSynthesis.getVoices();
  const chosen=voices[voiceSelect.value]||voices[0];
  for(let s of syllables){
    const textToSpeak=s.replace(/-/g," ");
    const u=new SpeechSynthesisUtterance(textToSpeak);
    if(chosen) u.voice=chosen;
    u.rate=0.95;
    u.pitch=1.0;
    const p=new Promise(resolve=>{u.onend=resolve; u.onerror=resolve;});
    speechSynthesis.speak(u);
    await p;
    await new Promise(r=>setTimeout(r,120));
  }
}

playBtn.addEventListener("click",()=>{speechSynthesis.cancel(); playVox();});

/* render first */
render();
</script>
</body>
</html>
