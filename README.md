<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Monaco Tabs with Proper Numbering - Darker Theme</title>

<style>
html,body{
    margin:0;
    height:100%;
    /* 완전 검은색에 가깝게 변경 */
    background:#0a0a0a; 
    overflow:hidden;
}

#topbar{
    display:flex;
    /* 상단바 배경색을 더 어둡게 변경 */
    background:#0a0a0a; 
    height:30px;
    overflow:hidden;
}

#tabs{
    display:flex;
    align-items:center;
    flex:1;
    overflow-x:auto;
    scrollbar-width:none;
    flex-wrap:nowrap;
    /* 탭 영역 배경색을 더 어둡게 변경 */
    background:#0a0a0a; 
}
#tabs::-webkit-scrollbar{display:none}

/* ============ NEW TAB STYLE ============ */
.tab{
    width: 75px;
    display:flex;
    align-items:center;
    gap:6px;
    padding:0 10px;
    height:30px;
    /* 탭 배경색 (비활성)을 더 어둡게 변경 */
    background:#1c1c1c; 
    color:#aaaaaa; /* 비활성 탭 텍스트 색상 조정 */
    font-family:"Segoe UI", sans-serif;
    font-size:14px;
    cursor:pointer;
    flex:0 0 auto;
    position:relative;
    border-top-left-radius:8px;
    border-top-right-radius:8px;
    border-bottom-left-radius:0;
    border-bottom-right-radius:0;
    opacity:1;
    transform:none;
    transition:none;
    /* 탭 사이의 구분선을 위해 약간의 여백 추가 */
    margin-right: 1px; 
}

.tab.active{
    /* 활성 탭 배경색을 상단바와 대비되게 조정 */
    background:#121212; 
    color:white;
}

.tab.active::after{
    content:none; /* removed green underline */
}

/* remove all animations */
.tab.show,
.tab.removing{
    opacity:1 !important;
    transform:none !important;
    transition:none !important;
}

/* "+" button */
#addTab{
    display:flex;
    align-items:center;
    justify-content:center;
    background:transparent;
    color:#777777; /* 버튼 색상 조정 */
    font-size:12px;
    font-family:"Segoe UI", sans-serif;
    cursor:pointer;
    width:32px;
    height:29px;
    flex:0 0 auto;
    /* 상단바와 동일한 배경색을 유지 */
    background:#0a0a0a; 
}

#container{
    width:100%;
    height:calc(100% - 30px);
}
</style>

</head>
<body>
<div id="topbar">
    <div id="tabs">
        <div id="addTab">+</div>
    </div>
</div>

<div id="container"></div>

<script src="https://unpkg.com/monaco-editor@latest/min/vs/loader.js"></script>

<script>
require.config({paths:{'vs':'https://unpkg.com/monaco-editor@latest/min/vs'}})

let editor,
    tabs = document.getElementById("tabs"),
    addBtn = document.getElementById("addTab"),
    models = {},
    active = null;

let savedSession = JSON.parse(localStorage.getItem("monacoTabs")) || {tabs:[], active:null};

require(["vs/editor/editor.main"], function() {

    monaco.languages.setLanguageConfiguration("lua", {
        comments:{lineComment:"--",blockComment:["--[[","]]"]},
        brackets:[["{","}"],["[","]"],["(",")"]],
        autoClosingPairs:[
            {open:"{",close:"}"},
            {open:"[",close:"]"},
            {open:"(",close:")"},
            {open:"\"",close:"\""},
            {open:"'",close:"'"}
        ],
        surroundingPairs:[
            {open:"{",close:"}"},
            {open:"[",close:"]"},
            {open:"(",close:")"},
            {open:"\"",close:"\""},
            {open:"'",close:"'"}
        ]
    });

    monaco.languages.setMonarchTokensProvider("lua",{
        defaultToken:"",
        tokenPostfix:".lua",
        keywords:[
            "and","break","do","else","elseif","end","false","for","function","goto",
            "if","in","local","nil","not","or","repeat","return","then","true","until","while"
        ],
        tokenizer:{
            root:[
                [/--\[\[/,{token:"comment",next:"@blockcomment"}],
                [/--.*/,"comment"],
                [/[A-Za-z_][A-Za-z0-9_]*/,{
                    cases:{"@keywords":"keyword","@default":"identifier"}
                }],
                [/\d+(\.\d+)?([eE][\-+]?\d+)?/,"number"],
                [/"/,{token:"string.quote",next:"@dqstring"}],
                [/'/,{token:"string.quote",next:"@sqstring"}],
                [/[\[\]{}().,;:+\-*\/%#^!=<>|~]/,"delimiter"]
            ],
            blockcomment:[
                [/\]\]/,{token:"comment",next:"@pop"}],
                [/./,"comment"]
            ],
            dqstring:[
                [/[^\\"]+/,"string"],
                [/\\./,"string.escape"],
                [/"/,{token:"string.quote",next:"@pop"}]
            ],
            sqstring:[
                [/[^\\']+/,"string"],
                [/\\./,"string.escape"],
                [/'/,{token:"string.quote",next:"@pop"}]
            ]
        }
    });

    monaco.editor.defineTheme("luauGrayTheme",{
        base:"vs-dark",
        inherit:false,
        rules:[
            {token:"keyword",foreground:"FF7C40"},
            {token:"identifier",foreground:"e0e0e0"},
            {token:"number",foreground:"FFDE59"},
            {token:"string",foreground:"ADF195"},
            {token:"comment",foreground:"888888"},
            {token:"delimiter",foreground:"aaaaaa"}
        ],
        colors:{
            /* Monaco Editor의 배경색을 더 어둡게 조정 */
            "editor.background":"#121212", 
            "editor.foreground":"#ffffff"
        }
    });

    editor = monaco.editor.create(document.getElementById("container"),{
        value:"",
        language:"lua",
        theme:"luauGrayTheme",
        automaticLayout:true,
        fontSize:14,
        cursorBlinking:"blink",
        cursorSmoothCaretAnimation:true
    });

    if(savedSession.tabs.length>0){
        savedSession.tabs.forEach(t=>loadTab(t.name,t.code));
        if(savedSession.active) activate(savedSession.active);
    } else {
        newTab();
    }
});

/* ======================== SESSION SAVE ======================== */
function saveSession(){
    const tabList=[];
    document.querySelectorAll(".tab").forEach(t=>{
        const name=t.dataset.name;
        if(models[name]) tabList.push({name:name,code:models[name].getValue()});
    });
    localStorage.setItem("monacoTabs",JSON.stringify({tabs:tabList,active:active}));
}

function getNextTabNumber(){
    let nums=[];
    document.querySelectorAll(".tab").forEach(t=>{
        const m=t.dataset.name.match(/Script\s+(\d+)/);
        if(m) nums.push(parseInt(m[1]));
    });
    let i=1;
    while(nums.includes(i)) i++;
    return i;
}

/* ======================== LOAD TAB ======================== */
function loadTab(name, code){
    const tab=document.createElement("div");
    tab.className="tab";
    tab.dataset.name=name;

    const title=document.createElement("span");
    title.textContent=name;

    const closeBtn=document.createElement("span");
    closeBtn.textContent="×";
    closeBtn.style.marginLeft="6px";
    closeBtn.style.cursor="pointer";
    closeBtn.style.fontFamily="Segoe UI";
    closeBtn.style.fontSize="20px";
    closeBtn.onclick=(e)=>{
        e.stopPropagation();
        removeTab(name,tab);
    };

    tab.onclick=()=>activate(name);
    tab.appendChild(title);
    tab.appendChild(closeBtn);
    tabs.insertBefore(tab,addBtn);

    const model=monaco.editor.createModel(code || "-- "+name,"lua");
    models[name]=model;
}

/* ======================== NEW TAB ======================== */
function newTab(){
    const name="Script "+getNextTabNumber();
    loadTab(name,"");
    activate(name);
    saveSession();
}

/* ======================== ACTIVATE ======================== */
function activate(name){
    active=name;
    document.querySelectorAll(".tab").forEach(t=>t.classList.remove("active"));
    const el=document.querySelector('.tab[data-name="'+name+'"]');
    if(el) el.classList.add("active");
    editor.setModel(models[name]);
    saveSession();
}

/* ======================== REMOVE TAB ======================== */
function removeTab(name, tab){
    if(models[name]) models[name].dispose();
    delete models[name];
    tab.remove();

    if(active===name){
        const first=document.querySelector(".tab");
        if(first) activate(first.dataset.name);
        else editor.setModel(monaco.editor.createModel("","lua"));
    }
    saveSession();
}

addBtn.onclick=()=>newTab();

setInterval(()=>{
    if(active && models[active]) saveSession();
},1000);
</script>
</body>
</html>
