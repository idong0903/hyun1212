/**
 * 산자연중학교 컨텐츠부 포털 · Cloudflare Worker v6.0
 * - Liquid Glass UI + Sidebar Navigation
 * - 실시간 DM, 패스키 극대화, 푸시 알림
 * - Samsung Now Brief 스타일
 * - 풍부한 애니메이션
 */

const LOGO_URL         = "https://i.ibb.co/93GnzDL2/removebg-preview.png";
const RECAPTCHA_SITE   = "6LfdP5IsAAAAAGERFPYKFbUNGm6wkz7Lk0Xo8Gap";
const RECAPTCHA_SECRET = "6LfdP5IsAAAAAJgELPhWzi8ySe7vabm9Cf0jxIcs";
const COOKIE_NAME      = "sjy_session";

/* KV */
const _M = {};
async function kvGet(env, key, def) {
  const d = def !== undefined ? def : [];
  if (env.SJY_KV) {
    try { const v = await env.SJY_KV.get(key); return v ? JSON.parse(v) : d; }
    catch { return d; }
  }
  return _M[key] !== undefined ? JSON.parse(JSON.stringify(_M[key])) : d;
}
async function kvPut(env, key, val) {
  if (env.SJY_KV) await env.SJY_KV.put(key, JSON.stringify(val));
  else _M[key] = JSON.parse(JSON.stringify(val));
}

/* Util */
function esc(s) {
  return String(s ?? "")
    .split("&").join("&amp;")
    .split("<").join("&lt;")
    .split(">").join("&gt;")
    .split('"').join("&quot;")
    .split("'").join("&#39;");
}
async function hashPw(pw) {
  const buf = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(pw));
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2,"0")).join("");
}
function b64s(str) {
  return btoa(unescape(encodeURIComponent(str)))
    .split("+").join("-").split("/").join("_").split("=").join("");
}
function b64b(buf) {
  let s=""; const u=new Uint8Array(buf);
  for(let i=0;i<u.length;i++) s+=String.fromCharCode(u[i]);
  return btoa(s).split("+").join("-").split("/").join("_").split("=").join("");
}
function d64s(str) {
  return decodeURIComponent(escape(atob(str.split("-").join("+").split("_").join("/"))));
}
function d64b(str) {
  const bin=atob(str.split("-").join("+").split("_").join("/"));
  const u=new Uint8Array(bin.length);
  for(let i=0;i<bin.length;i++) u[i]=bin.charCodeAt(i);
  return u;
}
async function signToken(payload, secret) {
  const h = b64s(JSON.stringify({alg:"HS256",typ:"JWT"}));
  const b = b64s(JSON.stringify(payload));
  const key = await crypto.subtle.importKey("raw",new TextEncoder().encode(secret),{name:"HMAC",hash:"SHA-256"},false,["sign"]);
  const sig = await crypto.subtle.sign("HMAC",key,new TextEncoder().encode(h+"."+b));
  return h+"."+b+"."+b64b(sig);
}
async function verifyToken(token, secret) {
  try {
    const [h,b,s] = token.split(".");
    const key = await crypto.subtle.importKey("raw",new TextEncoder().encode(secret),{name:"HMAC",hash:"SHA-256"},false,["verify"]);
    const ok = await crypto.subtle.verify("HMAC",key,d64b(s),new TextEncoder().encode(h+"."+b));
    if (!ok) return null;
    const p = JSON.parse(d64s(b));
    if (p.exp && Date.now() > p.exp) return null;
    return p;
  } catch { return null; }
}
function getCookie(req, name) {
  const h = req.headers.get("Cookie")||"";
  const m = h.match(new RegExp("(?:^|; )"+name+"=([^;]*)"));
  return m ? decodeURIComponent(m[1]) : null;
}
async function getSession(req, env) {
  const t = getCookie(req, COOKIE_NAME);
  if (!t) return null;
  const payload = await verifyToken(t, env.JWT_SECRET||"sjy-dev-secret");
  if (!payload || !payload.sid) return null;
  const sessions=await kvGet(env,"sessions",[]);
  const rec=sessions.find(s=>s.sid===payload.sid && !s.revoked);
  if(!rec) return null;
  const users=await kvGet(env,"users");
  const user=users.find(u=>u.id===payload.id || u.username===payload.username);
  if(!user || user.blocked){ await revokeSession(env,payload.sid); return null; }
  rec.lastActiveAt=kstDateTimeStr();
  rec.role=user.role; rec.name=user.name; rec.username=user.username; rec.isAdmin=user.isAdmin===true||user.role==="admin";
  await kvPut(env,"sessions",sessions);
  return {id:user.id,username:user.username,name:user.name,grade:user.grade,classNum:user.classNum,role:user.role,isAdmin:rec.isAdmin,sid:payload.sid,blocked:!!user.blocked};
}
async function verifyCap(token) {
  try {
    const r = await fetch("https://www.google.com/recaptcha/api/siteverify",{
      method:"POST", headers:{"Content-Type":"application/x-www-form-urlencoded"},
      body: new URLSearchParams({secret:RECAPTCHA_SECRET, response:token})
    });
    return (await r.json()).success === true;
  } catch { return false; }
}
function isAdm(s) { return s && (s.role==="admin" || s.isAdmin===true); }
function kstNow(){ return new Date(Date.now()+9*60*60*1000); }
function kstDateStr(d){
  const x=d?new Date(d):kstNow();
  const y=x.getUTCFullYear(), m=String(x.getUTCMonth()+1).padStart(2,"0"), day=String(x.getUTCDate()).padStart(2,"0");
  return y+"-"+m+"-"+day;
}
function kstDateTimeStr(d){
  const x=d?new Date(d):kstNow();
  const y=x.getUTCFullYear(), m=String(x.getUTCMonth()+1).padStart(2,"0"), day=String(x.getUTCDate()).padStart(2,"0");
  const h=String(x.getUTCHours()).padStart(2,"0"), mi=String(x.getUTCMinutes()).padStart(2,"0"), s=String(x.getUTCSeconds()).padStart(2,"0");
  return y+"-"+m+"-"+day+" "+h+":"+mi+":"+s+" KST";
}
function nowGreeting() {
  const h = kstNow().getHours();
  if (h >= 4 && h < 6) return "좋은 여명입니다.";
  if (h >= 6 && h < 11) return "좋은 아침입니다.";
  if (h >= 11 && h < 13) return "좋은 정오입니다.";
  if (h >= 13 && h < 17) return "좋은 오후입니다.";
  if (h >= 17 && h < 19) return "좋은 일몰입니다.";
  return "좋은 저녁입니다.";
}
function randId(pfx){ return (pfx||"id")+"_"+Date.now().toString(36)+Math.random().toString(36).slice(2,10); }
function sha256Buf(buf){ return crypto.subtle.digest("SHA-256", buf); }
function bufEq(a,b){
  a=new Uint8Array(a); b=new Uint8Array(b);
  if(a.length!==b.length) return false;
  for(let i=0;i<a.length;i++) if(a[i]!==b[i]) return false;
  return true;
}
async function saveWebAuthnChallenge(env, payload){
  const rows=await kvGet(env,"webauthn_challenges",{});
  rows[payload.id]=payload;
  await kvPut(env,"webauthn_challenges",rows);
}
async function takeWebAuthnChallenge(env, id){
  const rows=await kvGet(env,"webauthn_challenges",{});
  const row=rows[id]||null;
  if(row) delete rows[id];
  await kvPut(env,"webauthn_challenges",rows);
  return row;
}
function parseUA(ua){
  ua=String(ua||"");
  let os="기타 OS", br="브라우저";
  if(ua.includes("Windows")) os="Windows";
  else if(ua.includes("Android")) os="Android";
  else if(ua.includes("iPhone")||ua.includes("iPad")) os="iOS";
  else if(ua.includes("Mac OS X")) os="macOS";
  else if(ua.includes("Linux")) os="Linux";
  if(ua.includes("Edg/")) br="Edge";
  else if(ua.includes("Whale/")) br="Whale";
  else if(ua.includes("Chrome/")) br="Chrome";
  else if(ua.includes("Safari/")&&!ua.includes("Chrome/")) br="Safari";
  else if(ua.includes("Firefox/")) br="Firefox";
  return os+" · "+br;
}
async function getUserPrefs(env, userId){ return await kvGet(env,"prefs_"+userId,{pushEnabled:true,dmEnabled:true,dmBlocked:false}); }
async function saveUserPrefs(env, userId, patch){
  const prefs=await getUserPrefs(env,userId); Object.assign(prefs,patch||{}); await kvPut(env,"prefs_"+userId,prefs); return prefs;
}
async function createSessionRecord(env, user, req){
  const sessions=await kvGet(env,"sessions",[]);
  const sid=randId("sid");
  const ip=req.headers.get("CF-Connecting-IP")||req.headers.get("X-Forwarded-For")||"알 수 없음";
  const ua=req.headers.get("User-Agent")||"";
  sessions.unshift({sid,userId:user.id,username:user.username,name:user.name,role:user.role,isAdmin:user.isAdmin===true||user.role==="admin",ip,ua,device:parseUA(ua),loginAt:kstDateTimeStr(),lastActiveAt:kstDateTimeStr(),revoked:false});
  if(sessions.length>500) sessions.splice(500);
  await kvPut(env,"sessions",sessions);
  return sid;
}
async function revokeSession(env, sid){
  const sessions=await kvGet(env,"sessions",[]);
  let changed=false;
  sessions.forEach(s=>{ if(s.sid===sid){ s.revoked=true; changed=true; }});
  if(changed) await kvPut(env,"sessions",sessions);
}
async function revokeUserSessions(env, userId){
  const sessions=await kvGet(env,"sessions",[]);
  let changed=false;
  sessions.forEach(s=>{ if(s.userId===userId && !s.revoked){ s.revoked=true; changed=true; }});
  if(changed) await kvPut(env,"sessions",sessions);
}

const ROLE_ORDER = ["student","member","staff","teacher","bujang","faculty","admin"];
const ROLE_LBL   = {student:"🎓 학생",member:"👤 멤버",staff:"📸 부서원",teacher:"👩‍🏫 담당선생님",bujang:"👑 부장",faculty:"🏫 교직원",admin:"🛡️ 관리자"};
function rlbl(r){ return ROLE_LBL[r]||r; }
function rlvl(r){ return ROLE_ORDER.indexOf(r); }

async function ensureAdmin(env) {
  const users = await kvGet(env,"users");
  if (!users.find(u => u.username==="admin")) {
    users.unshift({id:"admin",username:"admin",password:await hashPw("admin"),name:"관리자",grade:0,classNum:0,attendanceNum:0,phone:"",role:"admin",isAdmin:true,createdAt:kstDateTimeStr()});
    await kvPut(env,"users",users);
  }
}

async function addNotif(env, uid, msg, type) {
  const notifs = await kvGet(env,"notifs_"+uid);
  notifs.unshift({id:Date.now().toString()+Math.random().toString(36).slice(2),msg,type:type||"info",read:false,at:kstDateTimeStr()});
  if (notifs.length > 60) notifs.splice(60);
  await kvPut(env,"notifs_"+uid,notifs);
}

async function checkAccess(env, session) {
  if (!session) return null;
  if (isAdm(session)) return null;
  const cfg = await kvGet(env,"config",{});
  if (cfg.blockAccess) return "block";
  if (cfg.maintenance) {
    if (cfg.maintStart && cfg.maintEnd) {
      const now = kstNow();
      const nowM = now.getUTCHours()*60+now.getUTCMinutes();
      const [sh,sm] = cfg.maintStart.split(":").map(Number);
      const [eh,em] = cfg.maintEnd.split(":").map(Number);
      const sM=sh*60+sm, eM=eh*60+em;
      const inRange = sM<=eM ? (nowM>=sM&&nowM<=eM) : (nowM>=sM||nowM<=eM);
      if (inRange) return "maint";
    } else return "maint";
  }
  return null;
}

const H = (b,s) => new Response(b, {status:s||200, headers:{"Content-Type":"text/html;charset=UTF-8"}});
const J = (d,s) => new Response(JSON.stringify(d), {status:s||200, headers:{"Content-Type":"application/json"}});
const REDIR = loc => new Response(null,{status:302,headers:{Location:loc}});

/* ── API: passkey delete ── */
async function apiPasskeyDelete(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  try{
    const {passkeyId}=await req.json();
    if(!passkeyId) return J({success:false,message:"패스키 ID가 필요합니다."});
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===s.id);
    if(idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    const before=(users[idx].passkeys||[]).length;
    users[idx].passkeys=(users[idx].passkeys||[]).filter(pk=>pk.id!==passkeyId);
    if(users[idx].passkeys.length===before) return J({success:false,message:"해당 패스키를 찾을 수 없습니다."});
    await kvPut(env,"users",users);
    return J({success:true});
  }catch(e){return J({success:false,message:"서버 오류"},500);}
}

/* ── API: passkey list ── */
async function apiPasskeyList(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const users=await kvGet(env,"users");
  const me=users.find(u=>u.id===s.id);
  if(!me) return J({success:false},404);
  return J({success:true,passkeys:(me.passkeys||[]).map(pk=>({id:pk.id,name:pk.name||"패스키",createdAt:pk.createdAt||""}))});
}

/* Router */
export default {
  async fetch(request, env) {
    await ensureAdmin(env);
    const url = new URL(request.url);
    const p   = url.pathname;
    const M   = request.method;

    if (M==="POST"   && p==="/api/login")               return apiLogin(request,env);
    if (M==="POST"   && p==="/api/passkey/login-options") return apiPasskeyLoginOptions(request,env);
    if (M==="POST"   && p==="/api/passkey/login-complete") return apiPasskeyLoginComplete(request,env);
    if (M==="POST"   && p==="/api/passkey/register-options") return apiPasskeyRegisterOptions(request,env);
    if (M==="POST"   && p==="/api/passkey/register-complete") return apiPasskeyRegisterComplete(request,env);
    if (M==="POST"   && p==="/api/passkey/delete") return apiPasskeyDelete(request,env);
    if (M==="GET"    && p==="/api/passkey/list") return apiPasskeyList(request,env);
    if (M==="POST"   && p==="/api/register")            return apiRegister(request,env);
    if (M==="POST"   && p==="/api/pw-hint")             return apiPwHint(request,env);
    if (M==="POST"   && p==="/api/pw-reset-self")       return apiPwResetSelf(request,env);
    if (M==="GET"    && p==="/api/pending")             return apiGetPending(request,env);
    if (M==="POST"   && p==="/api/approve")             return apiApprove(request,env);
    if (M==="DELETE" && p.startsWith("/api/pending/"))  return apiDenyPending(request,env,url);
    if (M==="GET"    && p==="/api/members")             return apiGetMembers(request,env);
    if (M==="DELETE" && p.startsWith("/api/member/"))   return apiDeleteMember(request,env,url);
    if (M==="POST"   && p==="/api/reset-pw")            return apiResetPw(request,env);
    if (M==="POST"   && p==="/api/update-user")         return apiUpdateUser(request,env);
    if (M==="POST"   && p==="/api/toggle-admin")        return apiToggleAdmin(request,env);
    if (M==="GET"    && p==="/api/notifs")              return apiGetNotifs(request,env);
    if (M==="POST"   && p==="/api/notif-read")          return apiNotifRead(request,env);
    if (M==="POST"   && p==="/api/notif-clear")         return apiNotifClear(request,env);
    if (M==="GET"    && p==="/api/schedules")           return apiGetSchedules(request,env);
    if (M==="POST"   && p==="/api/schedule")            return apiAddSchedule(request,env);
    if (M==="DELETE" && p.startsWith("/api/schedule/")) return apiDelSchedule(request,env,url);
    if (M==="GET"    && p==="/api/notices")             return apiGetNotices(request,env);
    if (M==="POST"   && p==="/api/notice")              return apiAddNotice(request,env);
    if (M==="DELETE" && p.startsWith("/api/notice/"))   return apiDelNotice(request,env,url);
    if (M==="GET"    && p==="/api/attendance")          return apiGetAttendance(request,env);
    if (M==="POST"   && p==="/api/attendance")          return apiAddAttendance(request,env);
    if (M==="DELETE" && p.startsWith("/api/attendance/"))return apiDelAttendance(request,env,url);
    if (M==="GET"    && p==="/api/drives")              return apiGetDrives(request,env);
    if (M==="POST"   && p==="/api/drive")               return apiAddDrive(request,env);
    if (M==="DELETE" && p.startsWith("/api/drive/"))    return apiDelDrive(request,env,url);
    if (M==="GET"    && p==="/api/config")              return apiGetConfig(request,env);
    if (M==="POST"   && p==="/api/config")              return apiSaveConfig(request,env);
    if (M==="POST"   && p==="/api/update-myinfo")       return apiUpdateMyInfo(request,env);
    if (M==="GET"    && p==="/api/me")                  return apiMe(request,env);
    if (M==="GET"    && p==="/api/my-sessions")         return apiMySessions(request,env);
    if (M==="POST"   && p==="/api/logout-session")      return apiLogoutSession(request,env);
    if (M==="POST"   && p==="/api/logout-all")          return apiLogoutAll(request,env);
    if (M==="POST"   && p==="/api/beacon-logout")       return apiBeaconLogout(request,env);
    if (M==="GET"    && p==="/api/online-sessions")     return apiOnlineSessions(request,env);
    if (M==="POST"   && p==="/api/force-logout")        return apiForceLogout(request,env);
    if (M==="POST"   && p==="/api/bulk-deny")           return apiBulkDeny(request,env);
    if (M==="POST"   && p==="/api/toggle-block-user")   return apiToggleBlockUser(request,env);
    if (M==="POST"   && p==="/api/save-prefs")          return apiSavePrefs(request,env);
    if (M==="GET"    && p==="/api/dm-users")            return apiDmUsers(request,env);
    if (M==="GET"    && p==="/api/dms")                 return apiGetDms(request,env);
    if (M==="POST"   && p==="/api/dms")                 return apiSendDm(request,env);
    if (p==="/auth/logout")                              return authLogout(request,env);

    const session = await getSession(request,env);
    if (p!=="/pw-find" && p!=="/register" && p!=="/" && p!=="") {
      const block = await checkAccess(env, session);
      if (block==="block") return H(pageMaint("block"));
      if (block==="maint") return H(pageMaint("maint"));
    }
    if (p==="/"||p==="") { if (session) return REDIR("/main"); return H(pageLogin()); }
    if (p==="/register") return H(pageRegister());
    if (p==="/pw-find")  return H(pagePwFind());
    if (p==="/main") { if (!session) return REDIR("/"); return H(pageMain(session)); }
    if (p==="/admin") { if (!session||!isAdm(session)) return REDIR("/"); return H(pageAdmin(session)); }
    return new Response("Not Found",{status:404});
  }
};

/* ═══════════════ ALL API HANDLERS ═══════════════ */
async function issueLoginResponse(env, req, user){
  const ia=user.role==="admin"||user.isAdmin===true;
  const sid=await createSessionRecord(env,user,req);
  const token=await signToken({id:user.id,username:user.username,name:user.name,grade:user.grade,classNum:user.classNum,role:user.role,isAdmin:ia,sid:sid,exp:Date.now()+86400_000},env.JWT_SECRET||"sjy-dev-secret");
  return new Response(JSON.stringify({success:true,role:user.role,isAdmin:ia}),{status:200,headers:{"Content-Type":"application/json","Set-Cookie":COOKIE_NAME+"="+encodeURIComponent(token)+"; Path=/; HttpOnly; SameSite=Lax; Max-Age=86400"}});
}

async function apiLogin(req,env){
  try {
    const {username,password}=await req.json();
    if (!username||!password) return J({success:false,message:"아이디와 비밀번호를 입력하세요."});
    const users=await kvGet(env,"users");
    const user=users.find(u=>u.username===username);
    if (!user||user.password!==await hashPw(password)) return J({success:false,message:"아이디 또는 비밀번호가 올바르지 않습니다."});
    if(user.blocked) return J({success:false,message:"차단된 계정입니다."});
    return await issueLoginResponse(env,req,user);
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiPasskeyLoginOptions(req,env){
  try{
    const {username}=await req.json();
    if(!username) return J({success:false,message:"아이디를 입력하세요."});
    const users=await kvGet(env,"users");
    const user=users.find(u=>u.username===username);
    if(!user || user.blocked) return J({success:false,message:"사용자를 찾을 수 없거나 차단된 계정입니다."});
    const passkeys=(user.passkeys||[]).filter(Boolean);
    if(!passkeys.length) return J({success:false,message:"등록된 패스키가 없습니다."});
    const challengeId=randId("wauth");
    const challenge=b64b(crypto.getRandomValues(new Uint8Array(32)));
    await saveWebAuthnChallenge(env,{id:challengeId,type:"login",userId:user.id,username:user.username,challenge,exp:Date.now()+5*60*1000});
    return J({success:true,challengeId,publicKey:{challenge,timeout:60000,userVerification:"preferred",rpId:new URL(req.url).hostname,allowCredentials:passkeys.map(pk=>({id:pk.id,type:"public-key"}))}});
  }catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiPasskeyLoginComplete(req,env){
  try{
    const body=await req.json();
    const row=await takeWebAuthnChallenge(env,body.challengeId);
    if(!row||row.type!=="login"||row.exp<Date.now()) return J({success:false,message:"인증이 만료되었습니다."});
    const users=await kvGet(env,"users");
    const user=users.find(u=>u.id===row.userId);
    if(!user||user.blocked) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    const pk=(user.passkeys||[]).find(x=>x.id===body.id);
    if(!pk) return J({success:false,message:"패스키 정보를 찾을 수 없습니다."});
    const clientDataJSON=d64b(body.response.clientDataJSON);
    const client=JSON.parse(new TextDecoder().decode(clientDataJSON));
    if(client.type!=="webauthn.get") return J({success:false,message:"잘못된 패스키 응답입니다."});
    if(client.challenge!==row.challenge) return J({success:false,message:"챌린지가 일치하지 않습니다."});
    const origin=new URL(req.url).origin;
    if(client.origin!==origin) return J({success:false,message:"Origin 검증에 실패했습니다."});
    const authData=d64b(body.response.authenticatorData);
    const sig=d64b(body.response.signature);
    const clientHash=await sha256Buf(clientDataJSON);
    const signed=new Uint8Array(authData.length+clientHash.byteLength);
    signed.set(authData,0); signed.set(new Uint8Array(clientHash),authData.length);
    const key=await crypto.subtle.importKey("spki",d64b(pk.publicKey),{name:"ECDSA",namedCurve:"P-256"},false,["verify"]);
    const ok=await crypto.subtle.verify({name:"ECDSA",hash:"SHA-256"},key,sig,signed);
    if(!ok) return J({success:false,message:"패스키 서명 검증에 실패했습니다."});
    return await issueLoginResponse(env,req,user);
  }catch(e){return J({success:false,message:"패스키 로그인 실패"},500);}
}
async function apiPasskeyRegisterOptions(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  try{
    const challengeId=randId("wauth");
    const challenge=b64b(crypto.getRandomValues(new Uint8Array(32)));
    await saveWebAuthnChallenge(env,{id:challengeId,type:"register",userId:s.id,challenge,exp:Date.now()+5*60*1000});
    const users=await kvGet(env,"users");
    const me=users.find(u=>u.id===s.id);
    const existing=(me&&me.passkeys||[]).map(pk=>({id:pk.id,type:"public-key"}));
    return J({success:true,challengeId,publicKey:{challenge,rp:{name:"산자연중학교 컨텐츠부 포털",id:new URL(req.url).hostname},user:{id:b64s(s.id),name:s.username,displayName:s.name},pubKeyCredParams:[{type:"public-key",alg:-7}],timeout:60000,attestation:"none",authenticatorSelection:{residentKey:"preferred",userVerification:"preferred"},excludeCredentials:existing}});
  }catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiPasskeyRegisterComplete(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  try{
    const body=await req.json();
    const row=await takeWebAuthnChallenge(env,body.challengeId);
    if(!row||row.type!=="register"||row.userId!==s.id||row.exp<Date.now()) return J({success:false,message:"등록이 만료되었습니다."});
    const clientDataJSON=d64b(body.response.clientDataJSON);
    const client=JSON.parse(new TextDecoder().decode(clientDataJSON));
    if(client.type!=="webauthn.create") return J({success:false,message:"잘못된 패스키 응답입니다."});
    if(client.challenge!==row.challenge) return J({success:false,message:"챌린지가 일치하지 않습니다."});
    const origin=new URL(req.url).origin;
    if(client.origin!==origin) return J({success:false,message:"Origin 검증에 실패했습니다."});
    if(!body.response.publicKey) return J({success:false,message:"이 브라우저는 패스키 공개키 추출을 지원하지 않습니다."});
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===s.id);
    if(idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    users[idx].passkeys=users[idx].passkeys||[];
    users[idx].passkeys.push({id:body.id,publicKey:body.response.publicKey,createdAt:kstDateTimeStr(),name:body.name||parseUA(req.headers.get("User-Agent")||"")});
    await kvPut(env,"users",users);
    return J({success:true});
  }catch(e){return J({success:false,message:"패스키 등록 실패"},500);}
}

async function apiRegister(req,env){
  try {
    const cfg=await kvGet(env,"config",{});
    if(cfg.registerClosed) return J({success:false,message:"현재 가입 신청을 받지 않고 있습니다."});
    const {name,grade,classNum,attendanceNum,phone,captchaToken,regType,subject,position,desiredUsername,desiredPassword}=await req.json();
    if (!name||!captchaToken) return J({success:false,message:"필수 항목을 입력하세요."});
    if (regType==="teacher") {
      if (!desiredUsername||!desiredUsername.trim()) return J({success:false,message:"사용할 아이디를 입력하세요."});
      if (!desiredPassword||!desiredPassword.trim()) return J({success:false,message:"사용할 비밀번호를 입력하세요."});
      const users=await kvGet(env,"users");
      if (users.find(u=>u.username===desiredUsername.trim())) return J({success:false,message:"이미 사용 중인 아이디입니다."});
      const pending=await kvGet(env,"pending");
      if (pending.find(p=>p.desiredUsername===desiredUsername.trim())) return J({success:false,message:"이미 신청된 아이디입니다."});
    } else {
      if (!phone) return J({success:false,message:"전화번호를 입력하세요."});
      const pending=await kvGet(env,"pending"), users=await kvGet(env,"users");
      if (pending.find(p=>p.phone===phone.trim())) return J({success:false,message:"이미 신청된 전화번호입니다."});
      if (users.find(u=>u.phone===phone.trim())) return J({success:false,message:"이미 가입된 전화번호입니다."});
    }
    if (!await verifyCap(captchaToken)) return J({success:false,message:"reCAPTCHA 인증에 실패했습니다."});
    const pending=await kvGet(env,"pending");
    const entry={id:Date.now().toString(),name:name.trim(),grade:Number(grade)||0,classNum:Number(classNum)||0,attendanceNum:Number(attendanceNum)||0,phone:(phone||"").trim(),regType:regType||"student",subject:subject||"",position:position||"",requestedAt:kstDateTimeStr(),ts:Date.now()};
    if (regType==="teacher") { entry.desiredUsername=(desiredUsername||"").trim(); entry.desiredPassword=await hashPw(desiredPassword); }
    pending.push(entry);
    await kvPut(env,"pending",pending);
    return J({success:true,message:"가입 신청이 완료되었습니다!\n관리자 승인 후 로그인이 가능합니다."});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiPwHint(req,env){
  try {
    const {username,hintPw,captchaToken}=await req.json();
    if (!username||!hintPw||!captchaToken) return J({success:false,message:"모든 항목을 입력하세요."});
    if (!await verifyCap(captchaToken)) return J({success:false,message:"reCAPTCHA 인증 실패."});
    const users=await kvGet(env,"users");
    const user=users.find(u=>u.username===username);
    if (!user) return J({success:false,message:"존재하지 않는 아이디입니다."});
    if (await hashPw(hintPw)===user.password) {
      const rt=await signToken({uid:user.id,type:"pwreset",exp:Date.now()+600_000},env.JWT_SECRET||"sjy-dev-secret");
      return J({success:true,match:true,resetToken:rt});
    }
    return J({success:true,match:false,message:"비밀번호가 일치하지 않습니다."});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiPwResetSelf(req,env){
  try {
    const {resetToken,newPassword,captchaToken}=await req.json();
    if (!resetToken||!newPassword||!captchaToken) return J({success:false,message:"모든 항목을 입력하세요."});
    if (!await verifyCap(captchaToken)) return J({success:false,message:"reCAPTCHA 인증 실패."});
    const payload=await verifyToken(resetToken,env.JWT_SECRET||"sjy-dev-secret");
    if (!payload||payload.type!=="pwreset") return J({success:false,message:"유효하지 않거나 만료된 요청입니다."});
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===payload.uid);
    if (idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    users[idx].password=await hashPw(newPassword);
    await kvPut(env,"users",users);
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiGetPending(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  return J({success:true,pending:await kvGet(env,"pending")});
}
async function apiApprove(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {pendingId,username,password,role,adminPw,captchaToken}=await req.json();
    if (!pendingId||!username?.trim()) return J({success:false,message:"아이디를 입력하세요."});
    if (!adminPw) return J({success:false,message:"관리자 비밀번호를 입력하세요."});
    const users=await kvGet(env,"users");
    const adm=users.find(u=>u.id===s.id);
    if (!adm||adm.password!==await hashPw(adminPw)) return J({success:false,message:"관리자 비밀번호가 올바르지 않습니다."});
    if (captchaToken && !await verifyCap(captchaToken)) return J({success:false,message:"reCAPTCHA 인증 실패."});
    const pending=await kvGet(env,"pending");
    const item=pending.find(p=>p.id===pendingId);
    if (!item) return J({success:false,message:"신청 항목을 찾을 수 없습니다."});
    if (users.find(u=>u.username===username.trim())) return J({success:false,message:"이미 사용 중인 아이디입니다."});
    const ar=role||"member";
    const pwHash=(password&&password.trim())?await hashPw(password):item.desiredPassword||await hashPw(password||"temp1234");
    const nu={id:Date.now().toString(),username:username.trim(),password:pwHash,name:item.name,grade:item.grade,classNum:item.classNum,attendanceNum:item.attendanceNum,phone:item.phone,role:ar,isAdmin:ar==="admin",regType:item.regType||"student",createdAt:kstDateTimeStr()};
    users.push(nu);
    await kvPut(env,"users",users);
    await kvPut(env,"pending",pending.filter(p=>p.id!==pendingId));
    await addNotif(env,nu.id,"🎉 가입이 승인되었습니다! 환영합니다.","success");
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiDenyPending(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {captchaToken,adminPw}=await req.json().catch(()=>({}));
    if(captchaToken && !await verifyCap(captchaToken)) return J({success:false,message:"reCAPTCHA 인증 실패."});
    const id=url.pathname.split("/api/pending/")[1];
    const pending=await kvGet(env,"pending");
    await kvPut(env,"pending",pending.filter(p=>p.id!==id));
    return J({success:true});
  } catch(e){
    const id=url.pathname.split("/api/pending/")[1];
    const pending=await kvGet(env,"pending");
    await kvPut(env,"pending",pending.filter(p=>p.id!==id));
    return J({success:true});
  }
}
async function apiGetMembers(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const users=await kvGet(env,"users");
  return J({success:true,members:users.map(u=>({...u,password:undefined,blocked:!!u.blocked}))});
}
async function apiDeleteMember(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const id=url.pathname.split("/api/member/")[1];
  if (id==="admin") return J({success:false,message:"기본 관리자 계정은 삭제할 수 없습니다."});
  const users=await kvGet(env,"users");
  await kvPut(env,"users",users.filter(u=>u.id!==id));
  return J({success:true});
}
async function apiResetPw(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {userId,newPassword}=await req.json();
    if (!userId||!newPassword?.trim()) return J({success:false,message:"새 비밀번호를 입력하세요."});
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===userId);
    if (idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    users[idx].password=await hashPw(newPassword);
    await kvPut(env,"users",users);
    await addNotif(env,userId,"🔑 비밀번호가 재설정되었습니다.","warn");
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiUpdateUser(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {userId,username,role,name}=await req.json();
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===userId);
    if (idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    if (username&&username!==users[idx].username) {
      if (users.find(u=>u.username===username&&u.id!==userId)) return J({success:false,message:"이미 사용 중인 아이디입니다."});
      users[idx].username=username;
    }
    if (role) users[idx].role=role;
    if (name) users[idx].name=name;
    await kvPut(env,"users",users);
    await addNotif(env,userId,"📝 계정 정보가 수정되었습니다.","info");
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiToggleAdmin(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {userId,isAdmin}=await req.json();
    if (userId==="admin") return J({success:false,message:"기본 관리자는 변경할 수 없습니다."});
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===userId);
    if (idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    users[idx].isAdmin=!!isAdmin;
    if (isAdmin&&users[idx].role!=="admin") users[idx].role="admin";
    else if (!isAdmin&&users[idx].role==="admin") users[idx].role="member";
    await kvPut(env,"users",users);
    await addNotif(env,userId,isAdmin?"🛡️ 관리자 권한이 부여되었습니다.":"👤 관리자 권한이 해제되었습니다.",isAdmin?"success":"warn");
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiGetNotifs(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const notifs=await kvGet(env,"notifs_"+s.id);
  return J({success:true,notifs,unread:notifs.filter(n=>!n.read).length});
}
async function apiNotifRead(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const notifs=await kvGet(env,"notifs_"+s.id);
  notifs.forEach(n=>n.read=true);
  await kvPut(env,"notifs_"+s.id,notifs);
  return J({success:true});
}
async function apiNotifClear(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  await kvPut(env,"notifs_"+s.id,[]);
  return J({success:true});
}
async function apiGetSchedules(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  return J({success:true,schedules:await kvGet(env,"schedules")});
}
async function apiAddSchedule(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {title,type,date,time,note,assignees}=await req.json();
    if (!title||!type||!date) return J({success:false,message:"제목, 유형, 날짜는 필수입니다."});
    const scheds=await kvGet(env,"schedules");
    scheds.unshift({id:Date.now().toString(),title,type,date,time:time||"",note:note||"",assignees:assignees||[],createdBy:s.name,createdAt:kstDateTimeStr()});
    await kvPut(env,"schedules",scheds);
    if (assignees&&assignees.length) {
      const tl={shoot:"📷 촬영",club:"🎯 동아리",meeting:"💼 회의"}[type]||type;
      for (const uid of assignees) await addNotif(env,uid,tl+" 일정 담당자로 지정: ["+title+"] "+date+(time?" "+time:""),type==="shoot"?"warn":"info");
    }
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiDelSchedule(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const id=url.pathname.split("/api/schedule/")[1];
  await kvPut(env,"schedules",(await kvGet(env,"schedules")).filter(x=>x.id!==id));
  return J({success:true});
}
async function apiGetNotices(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  return J({success:true,notices:await kvGet(env,"notices")});
}
async function apiAddNotice(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {title,content,noticeType}=await req.json();
    if (!title||!content) return J({success:false,message:"제목과 내용을 입력하세요."});
    const notices=await kvGet(env,"notices");
    notices.unshift({id:Date.now().toString(),title,content,noticeType:noticeType||"notice",author:s.name,createdAt:new Date().toLocaleString("ko-KR")});
    await kvPut(env,"notices",notices);
    const users=await kvGet(env,"users");
    const icon=noticeType==="update"?"🆕":"📢";
    for (const u of users) { if(u.id!==s.id) await addNotif(env,u.id,icon+" 새 "+(noticeType==="update"?"업데이트":"공지")+": "+title,"info"); }
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiDelNotice(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const id=url.pathname.split("/api/notice/")[1];
  await kvPut(env,"notices",(await kvGet(env,"notices")).filter(x=>x.id!==id));
  return J({success:true});
}
async function apiGetAttendance(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  return J({success:true,records:await kvGet(env,"attendance")});
}
async function apiAddAttendance(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {date,attendees,absentees,note}=await req.json();
    if (!date) return J({success:false,message:"날짜를 입력하세요."});
    const recs=await kvGet(env,"attendance");
    recs.unshift({id:Date.now().toString(),date,attendees:attendees||[],absentees:absentees||[],note:note||"",recordedBy:s.name,createdAt:new Date().toLocaleString("ko-KR")});
    await kvPut(env,"attendance",recs);
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiDelAttendance(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const id=url.pathname.split("/api/attendance/")[1];
  await kvPut(env,"attendance",(await kvGet(env,"attendance")).filter(x=>x.id!==id));
  return J({success:true});
}
async function apiGetDrives(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  return J({success:true,drives:await kvGet(env,"drives"),userRole:s.role});
}
async function apiAddDrive(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const {filename,description,url,minRole,filesize,category}=await req.json();
    if (!filename||!url) return J({success:false,message:"파일명과 URL은 필수입니다."});
    const drives=await kvGet(env,"drives");
    drives.unshift({id:Date.now().toString(),filename,description:description||"",url,minRole:minRole||"student",filesize:filesize||"",category:category||"기타",uploadedBy:s.name,uploadedAt:new Date().toLocaleString("ko-KR")});
    await kvPut(env,"drives",drives);
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiDelDrive(req,env,url){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const id=url.pathname.split("/api/drive/")[1];
  await kvPut(env,"drives",(await kvGet(env,"drives")).filter(x=>x.id!==id));
  return J({success:true});
}
async function apiGetConfig(req,env){
  const s=await getSession(req,env);
  const cfg=await kvGet(env,"config",{});
  if(isAdm(s)) return J({success:true,config:cfg});
  return J({success:true,config:{registerClosed:!!cfg.registerClosed}});
}
async function apiSaveConfig(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  try {
    const body=await req.json();
    const cfg=await kvGet(env,"config",{});
    Object.assign(cfg,body);
    await kvPut(env,"config",cfg);
    return J({success:true,config:cfg});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiUpdateMyInfo(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  try {
    const {currentPw,newPw,username}=await req.json();
    const users=await kvGet(env,"users");
    const idx=users.findIndex(u=>u.id===s.id);
    if (idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
    if (users[idx].password!==await hashPw(currentPw)) return J({success:false,message:"현재 비밀번호가 올바르지 않습니다."});
    if (newPw&&newPw.trim()) users[idx].password=await hashPw(newPw);
    if (username&&username.trim()&&username!==users[idx].username) {
      if (users.find(u=>u.username===username&&u.id!==s.id)) return J({success:false,message:"이미 사용 중인 아이디입니다."});
      users[idx].username=username;
    }
    await kvPut(env,"users",users);
    return J({success:true});
  } catch(e){return J({success:false,message:"서버 오류"},500);}
}
async function apiMe(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const access=await checkAccess(env,s);
  const users=await kvGet(env,"users"); const me=users.find(u=>u.id===s.id); if(!me) return J({success:false},404);
  const prefs=await getUserPrefs(env,s.id);
  const notices=await kvGet(env,"notices"); const schedules=await kvGet(env,"schedules"); const notifs=await kvGet(env,"notifs_"+s.id);
  return J({success:true,access:access||"ok",user:{id:me.id,username:me.username,name:me.name,role:me.role,isAdmin:me.isAdmin===true||me.role==="admin",blocked:!!me.blocked,passkeys:(me.passkeys||[]).length},prefs:prefs,counts:{notices:notices.length,schedules:schedules.length,unread:notifs.filter(n=>!n.read).length}});
}
async function apiMySessions(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const sessions=(await kvGet(env,"sessions",[])).filter(x=>x.userId===s.id && !x.revoked);
  return J({success:true,sessions:sessions});
}
async function apiLogoutSession(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const {sid}=await req.json(); if(!sid) return J({success:false,message:"세션이 없습니다."});
  await revokeSession(env,sid);
  return J({success:true,self:s.sid===sid});
}
async function apiLogoutAll(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  await revokeUserSessions(env,s.id);
  return J({success:true});
}
async function apiBeaconLogout(req,env){
  const s=await getSession(req,env); if(!s) return J({success:true});
  await revokeSession(env,s.sid);
  return J({success:true});
}
async function apiOnlineSessions(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const sessions=(await kvGet(env,"sessions",[])).filter(x=>!x.revoked);
  return J({success:true,sessions:sessions});
}
async function apiForceLogout(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const {sid,userId}=await req.json();
  if(sid) await revokeSession(env,sid);
  if(userId) await revokeUserSessions(env,userId);
  return J({success:true});
}
async function apiBulkDeny(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const {ids}=await req.json(); const pending=await kvGet(env,"pending");
  await kvPut(env,"pending",pending.filter(p=>!(ids||[]).includes(p.id)));
  return J({success:true});
}
async function apiToggleBlockUser(req,env){
  const s=await getSession(req,env); if(!isAdm(s)) return J({success:false},403);
  const {userId,blocked}=await req.json();
  const users=await kvGet(env,"users"); const idx=users.findIndex(u=>u.id===userId);
  if(idx<0) return J({success:false,message:"사용자를 찾을 수 없습니다."});
  users[idx].blocked=!!blocked;
  await kvPut(env,"users",users);
  if(blocked) await revokeUserSessions(env,userId);
  return J({success:true});
}
async function apiSavePrefs(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const prefs=await saveUserPrefs(env,s.id,await req.json());
  return J({success:true,prefs});
}
async function apiDmUsers(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const users=await kvGet(env,"users");
  return J({success:true,users:users.filter(u=>u.id!==s.id&&!u.blocked).map(u=>({id:u.id,name:u.name,role:u.role}))});
}
async function apiGetDms(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const url=new URL(req.url); const withId=url.searchParams.get("with");
  const dms=await kvGet(env,"dms",[]);
  return J({success:true,messages:dms.filter(m=>(m.fromId===s.id&&m.toId===withId)||(m.fromId===withId&&m.toId===s.id)).slice(-100)});
}
async function apiSendDm(req,env){
  const s=await getSession(req,env); if(!s) return J({success:false},401);
  const {toId,text}=await req.json();
  if(!toId||!text||!text.trim()) return J({success:false,message:"메시지를 입력하세요."});
  const users=await kvGet(env,"users"); const target=users.find(u=>u.id===toId);
  if(!target || target.blocked) return J({success:false,message:"상대방을 찾을 수 없습니다."});
  const prefs=await getUserPrefs(env,toId); if(prefs.dmBlocked) return J({success:false,message:"상대방이 DM을 받지 않도록 설정했습니다."});
  const dms=await kvGet(env,"dms",[]);
  dms.push({id:randId("dm"),fromId:s.id,fromName:s.name,toId:toId,text:text.trim(),at:kstDateTimeStr()});
  await kvPut(env,"dms",dms.slice(-500));
  await addNotif(env,toId,"💬 "+s.name+"님의 DM이 도착했습니다.","info");
  return J({success:true});
}
async function authLogout(req,env){
  const s=await getSession(req,env); if(s&&s.sid) await revokeSession(env,s.sid);
  return new Response(null,{status:302,headers:{Location:"/","Set-Cookie":COOKIE_NAME+"=; Path=/; HttpOnly; Max-Age=0"}});
}


/* ══════════════════════════════════════
   Global CSS - Liquid Glass + Animations
══════════════════════════════════════ */
const GCSS = `
*{margin:0;padding:0;box-sizing:border-box;-webkit-font-smoothing:antialiased;}
:root{
  --p:#1b4332;--a:#2d6a4f;--a2:#1a3325;--a3:#40916c;
  --lt:#d8f3dc;--bd:#b7dfc0;--bd2:#e8f5e9;
  --mu:#52796f;--bg:#f0f7f2;--card:#fff;
  --glass:rgba(255,255,255,.6);--glass-b:rgba(255,255,255,.35);
  --glass-border:rgba(185,219,190,.45);
  --sh:0 2px 18px rgba(27,67,50,.08);--sh2:0 8px 36px rgba(27,67,50,.13);
  --sh-glass:0 8px 32px rgba(27,67,50,.1);
}
body{font-family:'Noto Sans KR',sans-serif;background:var(--bg);color:var(--p);min-height:100vh;overflow-x:hidden;}
input,button,select,textarea{font-family:'Noto Sans KR',sans-serif;}
.form-row{margin-bottom:14px;}
.lbl{display:block;font-size:11px;font-weight:700;color:var(--mu);letter-spacing:1px;text-transform:uppercase;margin-bottom:6px;}
.inp{width:100%;padding:12px 14px;border:1.5px solid var(--bd);border-radius:13px;background:rgba(248,253,249,.8);backdrop-filter:blur(6px);font-size:14px;color:var(--p);outline:none;transition:all .3s cubic-bezier(.4,0,.2,1);}
.inp:focus{border-color:var(--a);background:#fff;box-shadow:0 0 0 4px rgba(45,106,79,.12);transform:translateY(-1px);}
.inp::placeholder{color:#a8c5b0;}
.pw-box{position:relative;}
.pw-box .inp{padding-right:72px;}
.pw-toggle{position:absolute;right:8px;top:50%;transform:translateY(-50%);border:none;background:#eef7f0;color:var(--mu);font-size:12px;font-weight:700;border-radius:10px;padding:7px 10px;cursor:pointer;transition:all .2s;}
.pw-toggle:hover{background:var(--lt);color:var(--a);}
.sel{appearance:none;background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%2352796f' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");background-repeat:no-repeat;background-position:right 14px center;}
.btn{display:inline-flex;align-items:center;justify-content:center;gap:6px;padding:11px 20px;border-radius:13px;border:none;font-size:14px;font-weight:700;cursor:pointer;transition:all .3s cubic-bezier(.4,0,.2,1);text-decoration:none;}
.bp{background:linear-gradient(135deg,var(--a),var(--a2));color:#fff;box-shadow:0 4px 14px rgba(45,106,79,.25);}
.bp:hover:not(:disabled){transform:translateY(-2px) scale(1.02);box-shadow:0 8px 22px rgba(45,106,79,.35);}
.bp:active:not(:disabled){transform:translateY(0) scale(.98);}
.bp:disabled{opacity:.5;cursor:not-allowed;}
.bg2{background:var(--glass);backdrop-filter:blur(10px);border:1.5px solid var(--glass-border);color:var(--mu);}
.bg2:hover{border-color:var(--a);color:var(--a);background:rgba(255,255,255,.8);}
.bdanger{background:rgba(255,245,245,.8);backdrop-filter:blur(6px);border:1.5px solid #fca5a5;color:#dc2626;}
.bdanger:hover{background:#fee2e2;}
.bsm{padding:7px 12px;font-size:12px;border-radius:10px;}
.bxs{padding:5px 9px;font-size:11px;border-radius:8px;}
.err{background:rgba(255,245,245,.9);border:1.5px solid #fca5a5;border-radius:12px;padding:11px 14px;font-size:13px;color:#dc2626;margin-bottom:12px;font-weight:500;display:none;animation:shake .4s ease;}
.suc{background:rgba(240,255,244,.9);border:1.5px solid #86efac;border-radius:12px;padding:14px;font-size:13px;color:#166534;text-align:center;line-height:1.7;display:none;}
@keyframes shake{0%,100%{transform:translateX(0);}20%,60%{transform:translateX(-4px);}40%,80%{transform:translateX(4px);}}
.toast-wrap{position:fixed;bottom:24px;right:18px;z-index:99998;display:flex;flex-direction:column;gap:9px;align-items:flex-end;}
.toast{padding:12px 16px;border-radius:14px;font-size:13px;font-weight:600;box-shadow:0 8px 28px rgba(0,0,0,.15);animation:slideInRight .4s cubic-bezier(.16,1,.3,1) both;max-width:300px;display:flex;align-items:center;gap:9px;backdrop-filter:blur(14px);}
.ts{background:linear-gradient(135deg,rgba(22,101,52,.9),rgba(20,83,45,.9));color:#fff;}
.te{background:linear-gradient(135deg,rgba(153,27,27,.9),rgba(127,29,29,.9));color:#fff;}
.ti{background:rgba(26,51,37,.9);color:#fff;}
@keyframes slideInRight{0%{opacity:0;transform:translateX(40px) scale(.9);}100%{opacity:1;transform:translateX(0) scale(1);}}
@keyframes fadeInUp{0%{opacity:0;transform:translateY(20px);}100%{opacity:1;transform:translateY(0);}}
@keyframes fadeInScale{0%{opacity:0;transform:scale(.92);}100%{opacity:1;transform:scale(1);}}
@keyframes slideInLeft{0%{opacity:0;transform:translateX(-30px);}100%{opacity:1;transform:translateX(0);}}
@keyframes float{0%,100%{transform:translateY(0);}50%{transform:translateY(-8px);}}
@keyframes pulse{0%,100%{opacity:1;}50%{opacity:.6;}}
@keyframes shimmer{0%{background-position:-200% 0;}100%{background-position:200% 0;}}
@keyframes glow{0%,100%{box-shadow:0 0 5px rgba(45,106,79,.3);}50%{box-shadow:0 0 20px rgba(45,106,79,.5);}}
@keyframes ripple{0%{transform:scale(0);opacity:.6;}100%{transform:scale(4);opacity:0;}}
.anim-fadeInUp{animation:fadeInUp .5s cubic-bezier(.16,1,.3,1) both;}
.anim-fadeInScale{animation:fadeInScale .4s cubic-bezier(.16,1,.3,1) both;}
.anim-slideInLeft{animation:slideInLeft .4s cubic-bezier(.16,1,.3,1) both;}
.stagger>*{animation:fadeInUp .5s cubic-bezier(.16,1,.3,1) both;}
.stagger>*:nth-child(1){animation-delay:.05s;}
.stagger>*:nth-child(2){animation-delay:.1s;}
.stagger>*:nth-child(3){animation-delay:.15s;}
.stagger>*:nth-child(4){animation-delay:.2s;}
.stagger>*:nth-child(5){animation-delay:.25s;}
.stagger>*:nth-child(6){animation-delay:.3s;}
.stagger>*:nth-child(7){animation-delay:.35s;}
.stagger>*:nth-child(8){animation-delay:.4s;}
.modal{position:fixed;inset:0;z-index:9999;background:rgba(0,0,0,.38);backdrop-filter:blur(12px);-webkit-backdrop-filter:blur(12px);display:none;align-items:center;justify-content:center;padding:14px;}
.modal.open{display:flex!important;animation:fadeIn .2s ease;}
@keyframes fadeIn{0%{opacity:0;}100%{opacity:1;}}
.mbox{background:rgba(255,255,255,.92);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-radius:22px;padding:24px 22px;max-width:440px;width:100%;box-shadow:0 20px 60px rgba(0,0,0,.15);animation:popIn .35s cubic-bezier(.16,1,.3,1) both;position:relative;max-height:90vh;overflow-y:auto;border:1px solid rgba(255,255,255,.5);}
@keyframes popIn{0%{opacity:0;transform:scale(.88) translateY(14px);}100%{opacity:1;transform:scale(1) translateY(0);}}
.m-x{position:absolute;top:12px;right:12px;width:28px;height:28px;border-radius:50%;background:rgba(240,247,242,.8);border:1.5px solid var(--bd);font-size:14px;cursor:pointer;display:flex;align-items:center;justify-content:center;transition:all .25s;color:var(--mu);}
.m-x:hover{transform:rotate(90deg) scale(1.1);background:var(--lt);}
.m-t{font-size:16px;font-weight:700;color:var(--p);margin-bottom:4px;}
.m-s{font-size:13px;color:var(--mu);margin-bottom:16px;line-height:1.55;}
.ipre{background:rgba(240,247,242,.7);border:1.5px solid var(--bd);border-radius:12px;padding:12px 14px;margin-bottom:16px;backdrop-filter:blur(6px);}
.ir{display:flex;gap:9px;margin-bottom:5px;font-size:13px;}
.ir:last-child{margin:0;}
.ik{font-weight:700;color:var(--mu);width:70px;flex-shrink:0;}
.iv{color:var(--p);font-weight:500;}
.toggle{position:relative;display:inline-block;width:42px;height:24px;flex-shrink:0;}
.toggle input{opacity:0;width:0;height:0;}
.tsl{position:absolute;inset:0;border-radius:12px;background:#d1fae5;border:1.5px solid var(--bd);cursor:pointer;transition:.35s cubic-bezier(.4,0,.2,1);}
.tsl::before{content:'';position:absolute;left:3px;top:3px;width:16px;height:16px;border-radius:50%;background:var(--bd);transition:.35s cubic-bezier(.4,0,.2,1);}
.toggle input:checked+.tsl{background:linear-gradient(135deg,var(--a),var(--a2));border-color:var(--a);}
.toggle input:checked+.tsl::before{transform:translateX(18px);background:#fff;}
.sjy-overlay{position:fixed;inset:0;z-index:99999;background:rgba(0,0,0,.45);backdrop-filter:blur(14px);-webkit-backdrop-filter:blur(14px);display:none;align-items:center;justify-content:center;padding:16px;}
.sjy-overlay.open{display:flex;animation:fadeIn .2s ease both;}
.sjy-dlg{background:rgba(255,255,255,.92);backdrop-filter:blur(24px);border-radius:22px;padding:30px 24px 22px;max-width:340px;width:100%;box-shadow:0 24px 64px rgba(0,0,0,.2);text-align:center;animation:popIn .28s cubic-bezier(.16,1,.3,1) both;border:1px solid rgba(255,255,255,.5);}
.sjy-dlg-ic{font-size:38px;margin-bottom:12px;animation:float 2s ease-in-out infinite;}
.sjy-dlg-title{font-size:16px;font-weight:800;color:var(--p);margin-bottom:6px;line-height:1.45;}
.sjy-dlg-sub{font-size:13px;color:var(--mu);margin-bottom:20px;line-height:1.6;}
.sjy-dlg-btns{display:flex;gap:9px;}
.tog-row{display:flex;align-items:center;justify-content:space-between;padding:12px 0;border-bottom:1.5px solid var(--bd2);}
.tog-row:last-child{border-bottom:none;}
.tog-l{font-size:13px;font-weight:600;color:var(--p);}
.tog-s{font-size:11px;color:var(--mu);margin-top:2px;}
.mem-chk-row{display:flex;align-items:center;gap:8px;padding:7px 10px;border-radius:9px;transition:all .2s;}
.mem-chk-row:hover{background:var(--lt);}
.mem-chk-row input{accent-color:var(--a);width:14px;height:14px;flex-shrink:0;}
.glass-card{background:var(--glass);backdrop-filter:blur(16px);-webkit-backdrop-filter:blur(16px);border:1px solid var(--glass-border);border-radius:18px;box-shadow:var(--sh-glass);transition:all .35s cubic-bezier(.4,0,.2,1);}
.glass-card:hover{transform:translateY(-2px);box-shadow:0 12px 40px rgba(27,67,50,.15);}
`;

function GHEAD(t){
  return '<!DOCTYPE html><html lang="ko"><head>\n<meta charset="UTF-8">\n<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,viewport-fit=cover">\n<title>'+t+' · 산자연중 컨텐츠부</title>\n<link rel="icon" type="image/png" href="'+LOGO_URL+'">\n<link rel="apple-touch-icon" href="'+LOGO_URL+'">\n<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;600;700;800&family=Montserrat:wght@700;800;900&display=swap" rel="stylesheet">\n<style>'+GCSS;
}

/* Common JS */
function commonJS() {
  return `
<script>
function e(s){
  s=String(s==null?'':s);
  s=s.split('&').join('&amp;');
  s=s.split('<').join('&lt;');
  s=s.split('>').join('&gt;');
  return s;
}
function toast(msg,t){
  var w=document.getElementById('tw');
  if(!w)return;
  var el=document.createElement('div');
  el.className='toast '+(t==='s'?'ts':t==='e'?'te':'ti');
  var ic=document.createElement('span');
  ic.style.fontSize='15px';
  ic.style.flexShrink='0';
  ic.textContent=(t==='s'?'\\u2705':t==='e'?'\\u274C':'\\u2139\\uFE0F');
  var tx=document.createElement('span');
  tx.textContent=msg;
  el.appendChild(ic);el.appendChild(tx);
  w.appendChild(el);
  setTimeout(function(){
    el.style.opacity='0';
    el.style.transform='translateY(8px) scale(.95)';
    el.style.transition='all .35s cubic-bezier(.4,0,.2,1)';
    setTimeout(function(){if(el.parentNode)el.remove();},350);
  },3200);
}
function oModal(id){var m=document.getElementById(id);if(m)m.classList.add('open');}
function cModal(id){var m=document.getElementById(id);if(m)m.classList.remove('open');}
function togglePw(id,btn){var el=document.getElementById(id);if(!el)return;var is=el.type==='password';el.type=is?'text':'password';if(btn)btn.textContent=is?'\\uC228\\uAE40':'\\uBCF4\\uAE30';}
function sjyConfirm(title,onOk,opts){
  opts=opts||{};
  var ov=document.getElementById('_sjy_ov');
  if(!ov){
    ov=document.createElement('div');
    ov.id='_sjy_ov';
    ov.className='sjy-overlay';
    document.body.appendChild(ov);
  }
  var dlg=document.createElement('div');
  dlg.className='sjy-dlg';
  var ic=document.createElement('div');
  ic.className='sjy-dlg-ic';
  ic.textContent=opts.danger?'\\u{1F5D1}\\uFE0F':'\\u2753';
  var tl=document.createElement('div');
  tl.className='sjy-dlg-title';
  tl.textContent=title;
  var sb=document.createElement('div');
  sb.className='sjy-dlg-sub';
  sb.textContent=opts.sub||'';
  sb.style.display=opts.sub?'block':'none';
  var btns=document.createElement('div');
  btns.className='sjy-dlg-btns';
  var bno=document.createElement('button');
  bno.className='btn bg2';
  bno.style.flex='1';
  bno.style.padding='12px 0';
  bno.textContent=opts.cancelText||'\\uCDE8\\uC18C';
  var bok=document.createElement('button');
  bok.className='btn '+(opts.danger?'bdanger':'bp');
  bok.style.flex='1';
  bok.style.padding='12px 0';
  bok.textContent=opts.okText||'\\uD655\\uC778';
  btns.appendChild(bno);btns.appendChild(bok);
  dlg.appendChild(ic);dlg.appendChild(tl);dlg.appendChild(sb);dlg.appendChild(btns);
  ov.innerHTML='';
  ov.appendChild(dlg);
  ov.classList.add('open');
  function close(){ov.classList.remove('open');}
  bno.onclick=close;
  ov.onclick=function(ev){if(ev.target===ov)close();};
  bok.onclick=function(){close();if(onOk)onOk();};
}
function sjyAlert(msg,opts){
  opts=opts||{};
  var ov=document.getElementById('_sjy_ov');
  if(!ov){ov=document.createElement('div');ov.id='_sjy_ov';ov.className='sjy-overlay';document.body.appendChild(ov);}
  var dlg=document.createElement('div');dlg.className='sjy-dlg';
  var ic=document.createElement('div');ic.className='sjy-dlg-ic';ic.textContent=opts.icon||'\\u2139\\uFE0F';
  var tl=document.createElement('div');tl.className='sjy-dlg-title';tl.textContent=msg;
  var btns=document.createElement('div');btns.className='sjy-dlg-btns';
  var bok=document.createElement('button');bok.className='btn bp';bok.style.flex='1';bok.style.padding='12px 0';bok.textContent='\\uD655\\uC778';
  btns.appendChild(bok);dlg.appendChild(ic);dlg.appendChild(tl);dlg.appendChild(btns);
  ov.innerHTML='';ov.appendChild(dlg);ov.classList.add('open');
  function close(){ov.classList.remove('open');}
  bok.onclick=function(){close();if(opts.onOk)opts.onOk();};
  ov.onclick=function(ev){if(ev.target===ov)close();};
}
function doLogout(){
  sjyConfirm('\\uB85C\\uADF8\\uC544\\uC6C3 \\uD558\\uC2DC\\uACA0\\uC2B5\\uB2C8\\uAE4C?',function(){
    location.href='/auth/logout';
  },{okText:'\\uB85C\\uADF8\\uC544\\uC6C3',cancelText:'\\uCDE8\\uC18C'});
}
document.addEventListener('keydown',function(k){
  if(k.key==='Escape'){
    document.querySelectorAll('.modal.open').forEach(function(m){m.classList.remove('open');});
    var ov=document.getElementById('_sjy_ov');if(ov)ov.classList.remove('open');
  }
});
document.addEventListener('click',function(ev){
  if(ev.target.classList.contains('modal'))ev.target.classList.remove('open');
});
function b64uToArr(s){var bin=atob(String(s).split('-').join('+').split('_').join('/'));var u=new Uint8Array(bin.length);for(var i=0;i<bin.length;i++)u[i]=bin.charCodeAt(i);return u;}
function arrToB64u(buf){var u=new Uint8Array(buf),s='';for(var i=0;i<u.length;i++)s+=String.fromCharCode(u[i]);return btoa(s).split('+').join('-').split('/').join('_').split('=')[0];}
</script>`;
}

/* ═══ pageMaint ═══ */
function pageMaint(type){
  const isMaint = type==="maint";
  return GHEAD(isMaint?"서버 점검 중":"접근 차단")+`
.page{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:20px;}
.orb{position:fixed;border-radius:50%;filter:blur(80px);pointer-events:none;animation:float 6s ease-in-out infinite;}
.o1{width:400px;height:400px;top:-100px;right:-100px;background:radial-gradient(circle,rgba(45,106,79,.13),transparent);}
.o2{width:300px;height:300px;bottom:-80px;left:-80px;background:radial-gradient(circle,rgba(27,67,50,.1),transparent);animation-delay:3s;}
.card{background:var(--glass);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-radius:24px;padding:44px 36px;max-width:420px;width:100%;text-align:center;box-shadow:0 20px 60px rgba(0,0,0,.1);border:1px solid rgba(255,255,255,.5);animation:fadeInScale .6s cubic-bezier(.16,1,.3,1) both;}
.ic{font-size:54px;margin-bottom:16px;animation:float 3s ease-in-out infinite;}
.tit{font-family:'Montserrat',sans-serif;font-size:24px;font-weight:900;margin-bottom:8px;}
.sub{font-size:14px;color:var(--mu);line-height:1.7;}
</style></head><body>
<div class="orb o1"></div><div class="orb o2"></div>
<div class="page"><div class="card">
  <div class="ic">${isMaint?"🔧":"🚫"}</div>
  <div class="tit" style="color:${isMaint?"var(--p)":"#dc2626"}">${isMaint?"서버 점검 중":"접근이 차단되었습니다"}</div>
  <div class="sub">${isMaint?"현재 서버 점검이 진행 중입니다.<br>불편을 드려 죄송합니다.":"현재 관리자에 의해 접근이 제한된 상태입니다.<br>관리자에게 문의하세요."}</div>
  <div style="margin-top:22px"><a href="/auth/logout" style="color:var(--mu);font-size:13px;text-decoration:none;font-weight:600;">← 로그아웃</a></div>
</div></div></body></html>`;
}


/* ═══ pageLogin ═══ */
function pageLogin(){
  return GHEAD("로그인")+`
.page{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:20px;overflow:hidden;}
.orb{position:fixed;border-radius:50%;filter:blur(80px);pointer-events:none;z-index:0;animation:float 8s ease-in-out infinite;}
.o1{width:500px;height:500px;top:-180px;right:-120px;background:radial-gradient(circle,rgba(45,106,79,.14),transparent);}
.o2{width:380px;height:380px;bottom:-100px;left:-100px;background:radial-gradient(circle,rgba(27,67,50,.1),transparent);animation-delay:4s;}
.wrap{position:relative;z-index:1;width:100%;max-width:400px;animation:fadeInUp .7s cubic-bezier(.16,1,.3,1) both;}
.logo-a{text-align:center;margin-bottom:22px;animation:fadeInScale .6s cubic-bezier(.16,1,.3,1) both;}
.logo-img{height:72px;object-fit:contain;filter:drop-shadow(0 4px 18px rgba(27,67,50,.2));transition:transform .4s;}.logo-img:hover{transform:scale(1.08) rotate(3deg);}
.school{font-size:18px;font-weight:800;color:var(--p);margin-top:10px;}
.dept{font-size:12px;color:var(--mu);margin-top:2px;}
.card{background:var(--glass);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-radius:24px;padding:30px 26px;box-shadow:0 12px 40px rgba(27,67,50,.1);border:1px solid rgba(255,255,255,.5);animation:fadeInUp .7s .1s cubic-bezier(.16,1,.3,1) both;}
.card-t{font-family:'Montserrat',sans-serif;font-size:22px;font-weight:900;color:var(--p);margin-bottom:3px;}
.card-s{font-size:13px;color:var(--mu);margin-bottom:22px;line-height:1.55;}
.divider{display:flex;align-items:center;gap:10px;margin:16px 0;}
.div-l{flex:1;height:1px;background:var(--bd);}
.div-t{font-size:12px;color:#aac8b2;white-space:nowrap;}
.help-row{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:8px;}
.hbtn{padding:11px;border-radius:12px;background:rgba(244,251,245,.7);backdrop-filter:blur(6px);border:1.5px solid var(--bd);color:var(--mu);font-size:13px;font-weight:600;cursor:pointer;transition:all .25s;text-align:center;text-decoration:none;display:block;}
.hbtn:hover{background:var(--lt);border-color:var(--a);color:var(--a);transform:translateY(-1px);}
.reg-btn{width:100%;padding:13px;background:linear-gradient(135deg,rgba(45,106,79,.08),rgba(27,67,50,.05));border:1.5px solid var(--bd);border-radius:13px;color:var(--a);font-size:14px;font-weight:700;cursor:pointer;transition:all .25s;}
.reg-btn:hover{background:var(--lt);border-color:var(--a);transform:translateY(-1px);}
.ft{text-align:center;margin-top:14px;font-size:11px;color:#aac8b2;animation:fadeInUp .7s .3s cubic-bezier(.16,1,.3,1) both;}
</style></head><body>
<div class="orb o1"></div><div class="orb o2"></div>
<div class="page"><div class="wrap">
  <div class="logo-a">
    <img src="${LOGO_URL}" class="logo-img" alt="" onerror="this.style.display='none'">
    <div class="school">산자연중학교</div>
    <div class="dept">컨텐츠부</div>
  </div>
  <div class="card">
    <div class="card-t">로그인</div>
    <div class="card-s">관리자로부터 발급받은 계정으로 로그인하세요.</div>
    <div class="err" id="err"></div>
    <form autocomplete="on">
    <div class="form-row"><label class="lbl">아이디</label><input class="inp" type="text" id="un" placeholder="아이디 입력" autocomplete="username"></div>
    <div class="form-row" style="margin-bottom:18px"><label class="lbl">비밀번호</label><div class="pw-box"><input class="inp" type="password" id="pw" placeholder="비밀번호 입력" autocomplete="current-password"><button type="button" class="pw-toggle" onclick="togglePw('pw',this)">보기</button></div></div>
    <button type="button" class="btn bp" style="width:100%;padding:14px;font-size:15px" id="btn-l" onclick="doLogin()">🔐 로그인</button>
    </form>
    <div class="divider"><div class="div-l"></div><div class="div-t">도움이 필요하신가요?</div><div class="div-l"></div></div>
    <div class="help-row">
      <a class="hbtn" href="/pw-find">🔑 비밀번호 찾기</a>
      <a class="hbtn" onclick="oModal('m-help')">❓ 계정 문의</a>
    </div>
    <button class="btn bg2" style="width:100%;margin-bottom:8px" onclick="doPasskeyLogin()">🔐 패스키로 로그인</button>
    <button class="reg-btn" onclick="location.href='/register'">📝 회원가입 신청하기</button>
  </div>
  <div class="ft">산자연중학교 컨텐츠부 · 2026</div>
</div></div>
<div class="modal" id="m-help">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-help')">✕</button>
    <div class="m-t">❓ 계정 도움말</div>
    <div style="font-size:13px;color:var(--mu);line-height:1.9;margin-bottom:14px">
      • <strong style="color:var(--p)">아이디를 모르는 경우</strong>: 관리자에게 직접 문의하세요.<br>
      • <strong style="color:var(--p)">비밀번호를 모르는 경우</strong>: 비밀번호 찾기를 이용하세요.<br>
      • <strong style="color:var(--p)">계정이 없는 경우</strong>: 회원가입 신청 후 관리자 승인을 기다리세요.
    </div>
    <div style="background:rgba(216,243,220,.6);backdrop-filter:blur(6px);border:1.5px solid var(--bd);border-radius:14px;padding:16px;text-align:center">
      <div style="font-size:13px;color:var(--p);font-weight:600;line-height:1.8">
        📞 관리자 문의<br>
        <strong style="font-size:15px;color:var(--a)">산자연중학교 컨텐츠부</strong><br>
        담당 선생님 또는 부서 관리자에게 문의하세요.
      </div>
    </div>
    <div style="display:flex;gap:8px;margin-top:14px">
      <a href="/pw-find" class="btn bp" style="flex:1">🔑 비밀번호 찾기</a>
      <button class="btn bg2" onclick="cModal('m-help')">닫기</button>
    </div>
  </div>
</div>
<div class="toast-wrap" id="tw"></div>
${commonJS()}
<script>
document.addEventListener('keydown',function(ev){if(ev.key==='Enter')doLogin();});
async function doPasskeyLogin(){
  if(!window.PublicKeyCredential){toast('이 브라우저는 패스키를 지원하지 않습니다.','e');return;}
  var un=document.getElementById('un').value.trim();
  if(!un){document.getElementById('err').textContent='패스키 로그인 전 아이디를 입력하세요.';document.getElementById('err').style.display='block';return;}
  try{
    var r=await fetch('/api/passkey/login-options',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:un})});
    var d=await r.json(); if(!d.success){toast(d.message||'패스키 로그인 준비 실패','e');return;}
    var pk=d.publicKey; pk.challenge=b64uToArr(pk.challenge); pk.allowCredentials=(pk.allowCredentials||[]).map(function(x){return {id:b64uToArr(x.id),type:x.type};});
    var cred=await navigator.credentials.get({publicKey:pk});
    var body={challengeId:d.challengeId,id:cred.id,response:{clientDataJSON:arrToB64u(cred.response.clientDataJSON),authenticatorData:arrToB64u(cred.response.authenticatorData),signature:arrToB64u(cred.response.signature)}};
    var r2=await fetch('/api/passkey/login-complete',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)});
    var d2=await r2.json(); if(d2.success){location.href='/main';} else toast(d2.message||'패스키 로그인 실패','e');
  }catch(ex){toast('패스키 로그인에 실패했습니다.','e');}
}
async function doLogin(){
  var un=document.getElementById('un').value.trim();
  var pw=document.getElementById('pw').value;
  var err=document.getElementById('err'),btn=document.getElementById('btn-l');
  err.style.display='none';
  if(!un||!pw){err.textContent='아이디와 비밀번호를 입력하세요.';err.style.display='block';return;}
  btn.disabled=true;btn.textContent='로그인 중...';
  try{
    var r=await fetch('/api/login',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:un,password:pw})});
    var d=await r.json();
    if(d.success){location.href='/main';}
    else{err.textContent=d.message||'로그인 실패';err.style.display='block';}
  }catch(ex){err.textContent='서버 오류가 발생했습니다.';err.style.display='block';}
  btn.disabled=false;btn.textContent='🔐 로그인';
}
</script></body></html>`;
}


function pagePwFind(){
  return GHEAD("비밀번호 찾기")+`
.page{min-height:100vh;padding:24px 16px 48px;display:flex;flex-direction:column;align-items:center;background:var(--bg);}
.orb{position:fixed;border-radius:50%;filter:blur(80px);pointer-events:none;z-index:0;}
.o1{width:400px;height:400px;top:-120px;right:-80px;background:radial-gradient(circle,rgba(45,106,79,.12),transparent);}
.wrap{position:relative;z-index:1;width:100%;max-width:420px;}
.back-a{display:inline-flex;align-items:center;gap:6px;padding:8px 13px;border-radius:12px;border:1.5px solid var(--bd);background:#fff;font-size:13px;font-weight:600;color:var(--mu);text-decoration:none;transition:all .2s;margin-bottom:16px;}
.back-a:hover{border-color:var(--a);color:var(--a);}
.card{background:var(--glass);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-radius:24px;border:1px solid rgba(255,255,255,.5);padding:28px 24px;box-shadow:0 12px 40px rgba(27,67,50,.1);border:1.5px solid rgba(185,219,190,.5);}
.step-ind{display:flex;gap:8px;margin-bottom:22px;}
.si-bar{height:4px;border-radius:2px;flex:1;background:var(--bd);transition:all .4s;}
.si-bar.done{background:linear-gradient(90deg,var(--a),var(--a3));}
.step{display:none;}
.step.active{display:block;}
</style>
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
</head><body>
<div class="orb o1"></div>
<div class="page"><div class="wrap">
  <a href="/" class="back-a">← 로그인으로</a>
  <div class="card">
    <div style="font-family:'Montserrat',sans-serif;font-size:21px;font-weight:900;color:var(--p);margin-bottom:4px">🔑 비밀번호 찾기</div>
    <div style="font-size:13px;color:var(--mu);margin-bottom:18px;line-height:1.6">기존 비밀번호로 본인 확인 후 재설정합니다.</div>
    <div class="step-ind">
      <div class="si-bar done" id="si1"></div>
      <div class="si-bar" id="si2"></div>
      <div class="si-bar" id="si3"></div>
    </div>
    <div class="step active" id="step1">
      <div class="err" id="err1"></div>
      <div class="form-row"><label class="lbl">아이디</label><input class="inp" type="text" id="s1un" placeholder="아이디 입력"></div>
      <div class="form-row"><label class="lbl">기억나는 비밀번호</label><div class="pw-box"><input class="inp" type="password" id="s1pw" placeholder="이전에 사용하던 비밀번호"><button type="button" class="pw-toggle" onclick="togglePw('s1pw',this)">보기</button></div></div>
      <div class="form-row">
        <span class="lbl">보안 인증</span>
        <div class="g-recaptcha" data-sitekey="${RECAPTCHA_SITE}" data-callback="onC1" data-expired-callback="onCE1"></div>
      </div>
      <button class="btn bp" style="width:100%" id="btn1" disabled onclick="doS1()">다음 단계 →</button>
    </div>
    <div class="step" id="step2">
      <div class="err" id="err2"></div>
      <div class="form-row"><label class="lbl">새 비밀번호</label><div class="pw-box"><input class="inp" type="password" id="s2pw" placeholder="새 비밀번호 (6자 이상)"><button type="button" class="pw-toggle" onclick="togglePw('s2pw',this)">보기</button></div></div>
      <div class="form-row"><label class="lbl">비밀번호 확인</label><div class="pw-box"><input class="inp" type="password" id="s2pw2" placeholder="새 비밀번호 재입력"><button type="button" class="pw-toggle" onclick="togglePw('s2pw2',this)">보기</button></div></div>
      <div class="form-row">
        <span class="lbl">보안 인증</span>
        <div class="g-recaptcha" data-sitekey="${RECAPTCHA_SITE}" data-callback="onC2" data-expired-callback="onCE2"></div>
      </div>
      <button class="btn bp" style="width:100%" id="btn2" disabled onclick="doS2()">비밀번호 변경</button>
    </div>
    <div class="step" id="step3">
      <div style="text-align:center;padding:20px 0">
        <div style="font-size:48px;margin-bottom:12px">✅</div>
        <div style="font-size:18px;font-weight:700;color:var(--a);margin-bottom:8px">변경 완료!</div>
        <div style="font-size:13px;color:var(--mu);margin-bottom:20px">새 비밀번호로 로그인하세요.</div>
        <a href="/" class="btn bp" style="width:100%">🔐 로그인하기</a>
      </div>
    </div>
  </div>
</div></div>
<div class="toast-wrap" id="tw"></div>
${commonJS()}
<script>
var cap1=null,cap2=null,rtok=null;
window.onC1=function(t){cap1=t;document.getElementById('btn1').disabled=false;}
window.onCE1=function(){cap1=null;document.getElementById('btn1').disabled=true;}
window.onC2=function(t){cap2=t;document.getElementById('btn2').disabled=false;}
window.onCE2=function(){cap2=null;document.getElementById('btn2').disabled=true;}
function goStep(n){
  document.querySelectorAll('.step').forEach(function(s){s.classList.remove('active');});
  document.getElementById('step'+n).classList.add('active');
  for(var i=1;i<=3;i++) document.getElementById('si'+i).classList.toggle('done',i<=n);
}
async function doS1(){
  var un=document.getElementById('s1un').value.trim(),pw=document.getElementById('s1pw').value;
  var err=document.getElementById('err1'),btn=document.getElementById('btn1');
  err.style.display='none';
  if(!un||!pw){err.textContent='아이디와 비밀번호를 입력하세요.';err.style.display='block';return;}
  if(!cap1){err.textContent='reCAPTCHA 인증을 완료하세요.';err.style.display='block';return;}
  btn.disabled=true;btn.textContent='확인 중...';
  try{
    var r=await fetch('/api/pw-hint',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:un,hintPw:pw,captchaToken:cap1})});
    var d=await r.json();
    if(d.success&&d.match){rtok=d.resetToken;goStep(2);}
    else{err.textContent=d.message||'비밀번호가 일치하지 않습니다.';err.style.display='block';if(window.grecaptcha)grecaptcha.reset();cap1=null;btn.disabled=true;}
  }catch(ex){err.textContent='서버 오류';err.style.display='block';}
  btn.textContent='다음 단계 →';btn.disabled=!cap1;
}
async function doS2(){
  var pw=document.getElementById('s2pw').value,pw2=document.getElementById('s2pw2').value;
  var err=document.getElementById('err2'),btn=document.getElementById('btn2');
  err.style.display='none';
  if(!pw||pw.length<6){err.textContent='비밀번호는 6자 이상이어야 합니다.';err.style.display='block';return;}
  if(pw!==pw2){err.textContent='비밀번호가 일치하지 않습니다.';err.style.display='block';return;}
  if(!cap2){err.textContent='reCAPTCHA 인증을 완료하세요.';err.style.display='block';return;}
  btn.disabled=true;btn.textContent='변경 중...';
  try{
    var r=await fetch('/api/pw-reset-self',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({resetToken:rtok,newPassword:pw,captchaToken:cap2})});
    var d=await r.json();
    if(d.success){goStep(3);}
    else{err.textContent=d.message||'변경 실패';err.style.display='block';if(window.grecaptcha)grecaptcha.reset();cap2=null;btn.disabled=true;}
  }catch(ex){err.textContent='서버 오류';err.style.display='block';}
  btn.textContent='비밀번호 변경';btn.disabled=!cap2;
}
</script></body></html>`;
}

/* ══════════════════════════════════════ pageRegister ══════════════════════════════════════ */
function pageRegister(){
  return GHEAD("회원가입 신청")+`
.page{min-height:100vh;padding:24px 16px 48px;display:flex;flex-direction:column;align-items:center;background:var(--bg);}
.orb{position:fixed;border-radius:50%;filter:blur(80px);pointer-events:none;z-index:0;}
.o1{width:420px;height:420px;top:-140px;right:-80px;background:radial-gradient(circle,rgba(45,106,79,.12),transparent);}
.o2{width:320px;height:320px;bottom:-80px;left:-80px;background:radial-gradient(circle,rgba(27,67,50,.08),transparent);}
.wrap{position:relative;z-index:1;width:100%;max-width:460px;}
.back-a{display:inline-flex;align-items:center;gap:6px;padding:8px 13px;border-radius:12px;border:1.5px solid var(--bd);background:#fff;font-size:13px;font-weight:600;color:var(--mu);text-decoration:none;transition:all .2s;margin-bottom:14px;}
.back-a:hover{border-color:var(--a);color:var(--a);}
.logo-a{text-align:center;margin-bottom:16px;}
.logo-img{height:52px;filter:drop-shadow(0 4px 12px rgba(27,67,50,.15));}
.card{background:var(--glass);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-radius:24px;border:1px solid rgba(255,255,255,.5);padding:26px 22px;box-shadow:0 12px 40px rgba(27,67,50,.1);border:1.5px solid rgba(185,219,190,.5);}
.notice{background:#fffbeb;border:1.5px solid #fde68a;border-radius:12px;padding:12px 14px;margin-bottom:16px;font-size:13px;color:#92400e;line-height:1.72;}
.type-sel{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:16px;}
.ts-btn{padding:14px;border-radius:14px;border:2px solid var(--bd);background:#f8fdf9;cursor:pointer;transition:all .22s;text-align:center;}
.ts-btn.active{border-color:var(--a);background:var(--lt);}
.ts-ic{font-size:22px;margin-bottom:4px;}
.ts-t{font-size:12px;font-weight:700;color:var(--mu);}
.ts-btn.active .ts-t{color:var(--a);}
.row2{display:grid;grid-template-columns:1fr 1fr;gap:10px;}
.sf,.tf{display:none;}
.sf.show,.tf.show{display:block;}
</style>
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
</head><body>
<div class="orb o1"></div><div class="orb o2"></div>
<div class="page"><div class="wrap">
  <a href="/" class="back-a">&larr; 로그인으로</a>
  <div class="logo-a">
    <img src="${LOGO_URL}" class="logo-img" alt="" onerror="this.style.display='none'">
    <div style="font-size:15px;font-weight:700;color:var(--p);margin-top:7px">산자연중학교 컨텐츠부</div>
  </div>
  <div class="card">
    <div style="font-family:'Montserrat',sans-serif;font-size:21px;font-weight:900;color:var(--p);margin-bottom:4px">회원가입 신청</div>
    <div style="font-size:13px;color:var(--mu);margin-bottom:16px;line-height:1.6">신청 후 관리자 검토를 통해 계정이 발급됩니다. 모든 시간은 KST 기준입니다.</div>
    <div class="notice" id="reg-closed-box" style="display:none;background:#fff5f5;border-color:#fca5a5;color:#b91c1c">&#x1F6AB; 현재 관리자가 가입 신청을 받지 않고 있습니다.</div>
    <div class="notice">&#x26A0;&#xFE0F; <strong>주의사항</strong>: 허위 정보 입력 시 승인이 거부됩니다. 전화번호는 확인 가능한 번호로 입력해주세요.</div>
    <div class="err" id="err"></div>
    <div class="suc" id="suc"></div>
    <div id="fb">
      <input type="hidden" id="rtype" value="student">
      <!-- 학생 가입 폼 -->
      <div id="student-form">
        <div class="form-row"><label class="lbl">이름 *</label><input class="inp" type="text" id="rname" placeholder="실명 (예: 홍길동)"></div>
        <div class="form-row"><label class="lbl">전화번호 *</label><input class="inp" type="tel" id="rphone" placeholder="예: 010-1234-5678"></div>
        <div class="row2">
          <div class="form-row"><label class="lbl">학년 *</label><select class="inp sel" id="rgrade"><option value="">선택</option><option value="1">1학년</option><option value="2">2학년</option><option value="3">3학년</option></select></div>
          <div class="form-row"><label class="lbl">반 *</label><select class="inp sel" id="rcls"><option value="">선택</option><option value="1">1반</option><option value="2">2반</option></select></div>
        </div>
        <div class="form-row"><label class="lbl">출석번호 *</label><input class="inp" type="number" id="ratt" placeholder="출석번호" min="1" max="40"></div>
      </div>
      <!-- 교사 가입 폼 -->
      <div id="teacher-form" style="display:none">
        <div class="form-row"><label class="lbl">이름 *</label><input class="inp" type="text" id="tname" placeholder="실명 (예: 홍길동)"></div>
        <div class="form-row"><label class="lbl">사용할 아이디 *</label><input class="inp" type="text" id="tusername" placeholder="사용할 아이디 입력"></div>
        <div class="form-row"><label class="lbl">사용할 비밀번호 *</label>
          <div class="pw-box"><input class="inp" type="password" id="tpassword" placeholder="사용할 비밀번호 입력"><button type="button" class="pw-toggle" onclick="togglePw('tpassword',this)">보기</button></div>
        </div>
      </div>
      <div class="form-row" style="margin-bottom:18px">
        <span class="lbl">보안 인증 (reCAPTCHA) *</span>
        <div class="g-recaptcha" data-sitekey="${RECAPTCHA_SITE}" data-callback="onCap" data-expired-callback="onCapE"></div>
      </div>
      <button class="btn bp" style="width:100%;padding:14px;font-size:15px" id="btn-sub" disabled onclick="handleSubmit()">&#x1F4DD; 회원가입 신청하기</button>
    </div>
  </div>
  <div style="text-align:center;margin-top:14px;font-size:11px;color:#aac8b2">산자연중학교 컨텐츠부 &middot; 2026</div>
</div></div>

<!-- 학생이신가요? 팝업 -->
<div class="modal" id="m-student-check">
  <div class="mbox" style="text-align:center">
    <button class="m-x" onclick="cModal('m-student-check')">&#x2715;</button>
    <div style="font-size:38px;margin-bottom:12px">&#x1F393;</div>
    <div class="m-t" style="font-size:18px">학생이신가요?</div>
    <div class="m-s">가입 유형을 확인합니다.</div>
    <div style="display:grid;gap:12px">
      <button class="btn bp" style="width:100%;padding:15px;font-size:15px;font-weight:800" id="student-confirm-btn" onclick="confirmStudent()" disabled>&#x1F393; 네, 학생입니다. (5)</button>
      <button style="background:none;border:none;color:#999;font-size:12px;cursor:pointer;padding:8px;text-decoration:underline" onclick="switchToTeacher()">아니오, 교사입니다.</button>
    </div>
  </div>
</div>

<!-- 교사인가요? 팝업 -->
<div class="modal" id="m-teacher-check">
  <div class="mbox" style="text-align:center">
    <button class="m-x" onclick="cModal('m-teacher-check')">&#x2715;</button>
    <div style="font-size:38px;margin-bottom:12px">&#x1F469;&#x200D;&#x1F3EB;</div>
    <div class="m-t" style="font-size:18px">교사이신가요?</div>
    <div class="m-s">교사 가입 유형을 확인합니다.</div>
    <div style="display:grid;gap:12px">
      <button class="btn bp" style="width:100%;padding:15px;font-size:15px;font-weight:800" id="teacher-confirm-btn" onclick="confirmTeacher()" disabled>&#x1F469;&#x200D;&#x1F3EB; 네, 교사입니다. (5)</button>
      <button style="background:none;border:none;color:#999;font-size:12px;cursor:pointer;padding:8px;text-decoration:underline" onclick="switchToStudent()">아니오, 학생입니다.</button>
    </div>
  </div>
</div>

<div class="toast-wrap" id="tw"></div>
${commonJS()}
<script>
var capTok=null;
var studentCountdown=null, studentRemain=5;
var teacherCountdown=null, teacherRemain=5;
var currentForm='student';

window.onCap=function(token){
  capTok=token;
  var btn=document.getElementById('btn-sub');
  var closed=document.getElementById('reg-closed-box');
  if(btn){
    if(closed&&closed.style.display==='block') btn.disabled=true;
    else btn.disabled=false;
  }
};
window.onCapE=function(){
  capTok=null;
  var btn=document.getElementById('btn-sub');
  if(btn) btn.disabled=true;
};

function showStudentForm(){
  currentForm='student';
  var sf=document.getElementById('student-form');
  var tf=document.getElementById('teacher-form');
  var rt=document.getElementById('rtype');
  if(sf) sf.style.display='block';
  if(tf) tf.style.display='none';
  if(rt) rt.value='student';
  var btn=document.getElementById('btn-sub');
  if(btn) btn.textContent='\u{1F4DD} \uD68C\uC6D0\uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
}
function showTeacherForm(){
  currentForm='teacher';
  var sf=document.getElementById('student-form');
  var tf=document.getElementById('teacher-form');
  var rt=document.getElementById('rtype');
  if(sf) sf.style.display='none';
  if(tf) tf.style.display='block';
  if(rt) rt.value='teacher';
  var btn=document.getElementById('btn-sub');
  if(btn) btn.textContent='\u{1F4DD} \uAD50\uC0AC \uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
}

function switchToTeacher(){
  cModal('m-student-check');
  clearInterval(studentCountdown);
  showTeacherForm();
  if(window.grecaptcha) try{ grecaptcha.reset(); }catch(ex){}
  capTok=null;
  var btn=document.getElementById('btn-sub');
  if(btn) btn.disabled=true;
}
function switchToStudent(){
  cModal('m-teacher-check');
  clearInterval(teacherCountdown);
  showStudentForm();
  if(window.grecaptcha) try{ grecaptcha.reset(); }catch(ex){}
  capTok=null;
  var btn=document.getElementById('btn-sub');
  if(btn) btn.disabled=true;
}

function handleSubmit(){
  var closed=document.getElementById('reg-closed-box');
  if(closed&&closed.style.display==='block'){ toast('\uD604\uC7AC \uAC00\uC785 \uC2E0\uCCAD\uC744 \uBC1B\uC9C0 \uC54A\uC2B5\uB2C8\uB2E4.','i'); return; }
  var err=document.getElementById('err');
  if(err){ err.style.display='none'; err.textContent=''; }

  if(currentForm==='student'){
    var nm=((document.getElementById('rname')||{}).value||'').trim();
    var ph=((document.getElementById('rphone')||{}).value||'').trim();
    var g=((document.getElementById('rgrade')||{}).value||'');
    var c=((document.getElementById('rcls')||{}).value||'');
    var a=((document.getElementById('ratt')||{}).value||'');
    if(!nm){ if(err){err.textContent='\uC774\uB984\uC744 \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!ph){ if(err){err.textContent='\uC804\uD654\uBC88\uD638\uB97C \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!g||!c||!a){ if(err){err.textContent='\uD559\uB144, \uBC18, \uCD9C\uC11D\uBC88\uD638\uB97C \uBAA8\uB450 \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!capTok){ if(err){err.textContent='reCAPTCHA \uC778\uC99D\uC744 \uC644\uB8CC\uD558\uC138\uC694.';err.style.display='block';} return; }
    openStudentCheck();
  } else {
    var tn=((document.getElementById('tname')||{}).value||'').trim();
    var tu=((document.getElementById('tusername')||{}).value||'').trim();
    var tp=((document.getElementById('tpassword')||{}).value||'');
    if(!tn){ if(err){err.textContent='\uC774\uB984\uC744 \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!tu){ if(err){err.textContent='\uC0AC\uC6A9\uD560 \uC544\uC774\uB514\uB97C \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!tp){ if(err){err.textContent='\uC0AC\uC6A9\uD560 \uBE44\uBC00\uBC88\uD638\uB97C \uC785\uB825\uD558\uC138\uC694.';err.style.display='block';} return; }
    if(!capTok){ if(err){err.textContent='reCAPTCHA \uC778\uC99D\uC744 \uC644\uB8CC\uD558\uC138\uC694.';err.style.display='block';} return; }
    openTeacherCheck();
  }
}

function openStudentCheck(){
  studentRemain=5;
  var btn=document.getElementById('student-confirm-btn');
  if(btn){
    btn.disabled=true;
    btn.innerHTML='\u{1F393} \uB124, \uD559\uC0DD\uC785\uB2C8\uB2E4. ('+studentRemain+')';
  }
  clearInterval(studentCountdown);
  studentCountdown=setInterval(function(){
    studentRemain--;
    if(btn){
      if(studentRemain>0) btn.innerHTML='\u{1F393} \uB124, \uD559\uC0DD\uC785\uB2C8\uB2E4. ('+studentRemain+')';
      else{
        btn.innerHTML='\u{1F393} \uB124, \uD559\uC0DD\uC785\uB2C8\uB2E4.';
        btn.disabled=false;
        clearInterval(studentCountdown);
      }
    }
  },1000);
  oModal('m-student-check');
}
function confirmStudent(){
  cModal('m-student-check');
  clearInterval(studentCountdown);
  doRegStudent();
}

function openTeacherCheck(){
  teacherRemain=5;
  var btn=document.getElementById('teacher-confirm-btn');
  if(btn){
    btn.disabled=true;
    btn.innerHTML='\u{1F469}\u200D\u{1F3EB} \uB124, \uAD50\uC0AC\uC785\uB2C8\uB2E4. ('+teacherRemain+')';
  }
  clearInterval(teacherCountdown);
  teacherCountdown=setInterval(function(){
    teacherRemain--;
    if(btn){
      if(teacherRemain>0) btn.innerHTML='\u{1F469}\u200D\u{1F3EB} \uB124, \uAD50\uC0AC\uC785\uB2C8\uB2E4. ('+teacherRemain+')';
      else{
        btn.innerHTML='\u{1F469}\u200D\u{1F3EB} \uB124, \uAD50\uC0AC\uC785\uB2C8\uB2E4.';
        btn.disabled=false;
        clearInterval(teacherCountdown);
      }
    }
  },1000);
  oModal('m-teacher-check');
}
function confirmTeacher(){
  cModal('m-teacher-check');
  clearInterval(teacherCountdown);
  doRegTeacher();
}

async function loadRegCfg(){
  try{
    var r=await fetch('/api/config');
    var d=await r.json();
    if(d&&d.success&&d.config&&d.config.registerClosed){
      var box=document.getElementById('reg-closed-box');
      var btn=document.getElementById('btn-sub');
      var fb=document.getElementById('fb');
      if(box) box.style.display='block';
      if(btn) btn.disabled=true;
      if(fb){
        fb.style.opacity='.55';
        fb.style.pointerEvents='none';
      }
    }
  }catch(ex){}
}

async function doRegStudent(){
  var nm=((document.getElementById('rname')||{}).value||'').trim();
  var ph=((document.getElementById('rphone')||{}).value||'').trim();
  var err=document.getElementById('err');
  var suc=document.getElementById('suc');
  var btn=document.getElementById('btn-sub');
  if(err){ err.style.display='none'; err.textContent=''; }
  if(suc){ suc.style.display='none'; suc.innerHTML=''; }
  if(btn){ btn.disabled=true; btn.textContent='\uC2E0\uCCAD \uC911...'; }
  try{
    var payload={
      name:nm,
      phone:ph,
      captchaToken:capTok,
      regType:'student',
      grade:(((document.getElementById('rgrade')||{}).value)||'0'),
      classNum:(((document.getElementById('rcls')||{}).value)||'0'),
      attendanceNum:(((document.getElementById('ratt')||{}).value)||'0')
    };
    var r=await fetch('/api/register',{
      method:'POST',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify(payload)
    });
    var d=await r.json();
    if(d&&d.success){
      var fb=document.getElementById('fb');
      if(fb) fb.style.display='none';
      if(suc){
        var msg=String(d.message||'\uAC00\uC785 \uC2E0\uCCAD\uC774 \uC644\uB8CC\uB418\uC5C8\uC2B5\uB2C8\uB2E4!');
        suc.innerHTML='<strong>\u2705 \uC2E0\uCCAD \uC644\uB8CC!</strong><br><br>'+msg.split(String.fromCharCode(10)).join('<br>')+'<br><br><a href="/" style="color:var(--a);font-weight:700">\u2190 \uB85C\uADF8\uC778 \uD398\uC774\uC9C0\uB85C</a>';
        suc.style.display='block';
      }
    }else{
      if(err){
        err.textContent=(d&&d.message)?d.message:'\uAC00\uC785 \uC2E0\uCCAD\uC5D0 \uC2E4\uD328\uD588\uC2B5\uB2C8\uB2E4.';
        err.style.display='block';
      }
      if(window.grecaptcha) try{ grecaptcha.reset(); }catch(e){}
      capTok=null;
      if(btn){
        btn.disabled=true;
        btn.textContent='\u{1F4DD} \uD68C\uC6D0\uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
      }
    }
  }catch(ex){
    console.error(ex);
    if(err){
      err.textContent='\uC694\uCCAD \uCC98\uB9AC \uC911 \uC624\uB958\uAC00 \uBC1C\uC0DD\uD588\uC2B5\uB2C8\uB2E4.';
      err.style.display='block';
    }
    if(btn){
      btn.disabled=false;
      btn.textContent='\u{1F4DD} \uD68C\uC6D0\uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
    }
  }
}

async function doRegTeacher(){
  var tn=((document.getElementById('tname')||{}).value||'').trim();
  var tu=((document.getElementById('tusername')||{}).value||'').trim();
  var tp=((document.getElementById('tpassword')||{}).value||'');
  var err=document.getElementById('err');
  var suc=document.getElementById('suc');
  var btn=document.getElementById('btn-sub');
  if(err){ err.style.display='none'; err.textContent=''; }
  if(suc){ suc.style.display='none'; suc.innerHTML=''; }
  if(btn){ btn.disabled=true; btn.textContent='\uC2E0\uCCAD \uC911...'; }
  try{
    var payload={
      name:tn,
      phone:'',
      captchaToken:capTok,
      regType:'teacher',
      grade:'0',
      classNum:'0',
      attendanceNum:'0',
      desiredUsername:tu,
      desiredPassword:tp
    };
    var r=await fetch('/api/register',{
      method:'POST',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify(payload)
    });
    var d=await r.json();
    if(d&&d.success){
      var fb=document.getElementById('fb');
      if(fb) fb.style.display='none';
      if(suc){
        var msg=String(d.message||'\uAC00\uC785 \uC2E0\uCCAD\uC774 \uC644\uB8CC\uB418\uC5C8\uC2B5\uB2C8\uB2E4!');
        suc.innerHTML='<strong>\u2705 \uC2E0\uCCAD \uC644\uB8CC!</strong><br><br>'+msg.split(String.fromCharCode(10)).join('<br>')+'<br><br><a href="/" style="color:var(--a);font-weight:700">\u2190 \uB85C\uADF8\uC778 \uD398\uC774\uC9C0\uB85C</a>';
        suc.style.display='block';
      }
    }else{
      if(err){
        err.textContent=(d&&d.message)?d.message:'\uAC00\uC785 \uC2E0\uCCAD\uC5D0 \uC2E4\uD328\uD588\uC2B5\uB2C8\uB2E4.';
        err.style.display='block';
      }
      if(window.grecaptcha) try{ grecaptcha.reset(); }catch(e){}
      capTok=null;
      if(btn){
        btn.disabled=true;
        btn.textContent='\u{1F4DD} \uAD50\uC0AC \uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
      }
    }
  }catch(ex){
    console.error(ex);
    if(err){
      err.textContent='\uC694\uCCAD \uCC98\uB9AC \uC911 \uC624\uB958\uAC00 \uBC1C\uC0DD\uD588\uC2B5\uB2C8\uB2E4.';
      err.style.display='block';
    }
    if(btn){
      btn.disabled=false;
      btn.textContent='\u{1F4DD} \uAD50\uC0AC \uAC00\uC785 \uC2E0\uCCAD\uD558\uAE30';
    }
  }
}

showStudentForm();
loadRegCfg();
</script></body></html>`;
}


/* ═══ pageMain - COMPLETELY REWRITTEN ═══ */
function pageMain(session){
  const isA = isAdm(session);
  const rl   = session.role||"member";
  const name = esc(session.name);
  return GHEAD("홈")+`
body{overflow-x:hidden;padding-bottom:env(safe-area-inset-bottom);background:var(--bg);}
.orb{position:fixed;border-radius:50%;filter:blur(80px);pointer-events:none;z-index:0;animation:float 8s ease-in-out infinite;}
.o1{width:550px;height:500px;top:-180px;right:-150px;background:rgba(45,106,79,.09);}
.o2{width:420px;height:420px;bottom:-130px;left:-120px;background:rgba(27,67,50,.07);animation-delay:4s;}

/* ── Sidebar (Desktop) ── */
.sidebar{position:fixed;left:0;top:0;bottom:0;width:240px;background:var(--glass);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);border-right:1px solid var(--glass-border);z-index:200;display:flex;flex-direction:column;transition:transform .35s cubic-bezier(.4,0,.2,1);overflow-y:auto;}
.sb-logo{display:flex;align-items:center;gap:10px;padding:18px 16px;border-bottom:1px solid var(--glass-border);}
.sb-logo img{height:28px;transition:transform .3s;}.sb-logo img:hover{transform:rotate(10deg) scale(1.1);}
.sb-logo-n{font-size:13px;font-weight:800;color:var(--p);}
.sb-user{display:flex;align-items:center;gap:10px;padding:14px 16px;margin:10px 8px;background:rgba(216,243,220,.5);border:1px solid var(--glass-border);border-radius:14px;animation:fadeInScale .5s both;}
.sb-avatar{width:36px;height:36px;border-radius:50%;background:linear-gradient(135deg,var(--a),var(--a2));display:flex;align-items:center;justify-content:center;font-size:14px;font-weight:700;color:#fff;flex-shrink:0;}
.sb-uname{font-size:12px;font-weight:700;color:var(--p);}
.sb-urole{font-size:10px;color:var(--a);font-weight:700;letter-spacing:.5px;}
.sb-nav{flex:1;padding:6px 8px;overflow-y:auto;}
.sb-section{font-size:10px;font-weight:700;letter-spacing:2px;color:#a8c5b0;text-transform:uppercase;padding:12px 10px 4px;}
.sb-item{display:flex;align-items:center;gap:9px;padding:10px 12px;border-radius:12px;font-size:13px;font-weight:500;color:var(--mu);cursor:pointer;transition:all .25s cubic-bezier(.4,0,.2,1);text-decoration:none;margin-bottom:2px;border:none;background:none;width:100%;text-align:left;position:relative;overflow:hidden;}
.sb-item::before{content:'';position:absolute;left:0;top:0;width:0;height:100%;background:linear-gradient(90deg,rgba(45,106,79,.1),transparent);transition:width .3s;border-radius:12px;}
.sb-item:hover{color:var(--p);transform:translateX(3px);}
.sb-item:hover::before{width:100%;}
.sb-item.active{background:linear-gradient(135deg,rgba(45,106,79,.12),rgba(27,67,50,.06));color:var(--a);font-weight:700;}
.sb-item.active::before{width:100%;}
.sb-ic{font-size:16px;width:20px;text-align:center;flex-shrink:0;}
.sb-badge{background:var(--a);color:#fff;border-radius:100px;font-size:10px;font-weight:700;padding:2px 7px;margin-left:auto;animation:pulse 2s infinite;}
.sb-foot{padding:12px;border-top:1px solid var(--glass-border);}
.sb-logout{display:flex;align-items:center;gap:8px;font-size:12px;color:var(--mu);text-decoration:none;padding:8px 12px;border-radius:10px;transition:all .2s;border:none;background:none;width:100%;cursor:pointer;}
.sb-logout:hover{background:rgba(255,245,245,.8);color:#dc2626;}

/* ── Hamburger (Mobile) ── */
.hamburger{display:none;position:fixed;top:12px;left:12px;z-index:300;width:40px;height:40px;border-radius:12px;background:var(--glass);backdrop-filter:blur(16px);border:1px solid var(--glass-border);cursor:pointer;flex-direction:column;align-items:center;justify-content:center;gap:4px;transition:all .3s;}
.hamburger span{display:block;width:18px;height:2px;background:var(--p);border-radius:2px;transition:all .3s cubic-bezier(.4,0,.2,1);}
.hamburger.open span:nth-child(1){transform:rotate(45deg) translate(4px,4px);}
.hamburger.open span:nth-child(2){opacity:0;transform:scaleX(0);}
.hamburger.open span:nth-child(3){transform:rotate(-45deg) translate(4px,-4px);}
.sb-overlay{display:none;position:fixed;inset:0;background:rgba(0,0,0,.3);z-index:190;backdrop-filter:blur(4px);}

/* ── Top Bar ── */
.topbar{position:sticky;top:0;z-index:100;height:58px;display:flex;align-items:center;justify-content:flex-end;padding:0 20px;background:var(--glass);backdrop-filter:blur(20px);-webkit-backdrop-filter:blur(20px);border-bottom:1px solid var(--glass-border);margin-left:240px;gap:10px;transition:margin-left .35s;}
.notif-btn{display:flex;align-items:center;gap:6px;position:relative;padding:7px 14px;border-radius:11px;border:1.5px solid var(--bd);background:rgba(255,255,255,.7);backdrop-filter:blur(6px);font-size:13px;font-weight:600;color:var(--mu);cursor:pointer;transition:all .25s;}
.notif-btn:hover{border-color:var(--a);color:var(--a);transform:translateY(-1px);}
.notif-ic{font-size:16px;}
.nbadge{position:absolute;top:-5px;right:-5px;min-width:18px;height:18px;background:#ef4444;border-radius:9px;font-size:10px;font-weight:700;color:#fff;display:none;align-items:center;justify-content:center;padding:0 4px;border:2px solid var(--bg);animation:pulse 2s infinite;}

/* ── Notification Panel ── */
.notif-panel{position:fixed;top:64px;right:14px;width:340px;max-height:460px;background:var(--glass);backdrop-filter:blur(24px);border:1px solid var(--glass-border);border-radius:18px;box-shadow:0 12px 40px rgba(27,67,50,.12);z-index:9990;display:none;flex-direction:column;overflow:hidden;}
.notif-panel.open{display:flex;animation:fadeInScale .25s cubic-bezier(.16,1,.3,1) both;}
.np-head{display:flex;align-items:center;justify-content:space-between;padding:13px 14px;border-bottom:1px solid var(--glass-border);}
.np-t{font-size:14px;font-weight:700;color:var(--p);}
.np-body{overflow-y:auto;flex:1;padding:8px;}
.nitem{padding:10px 12px;border-radius:10px;margin-bottom:4px;transition:all .2s;}
.nitem.unread{background:rgba(216,243,220,.5);}
.nitem-msg{font-size:13px;color:var(--p);line-height:1.45;margin-bottom:2px;}
.nitem-time{font-size:11px;color:var(--mu);}
.np-empty{text-align:center;padding:28px;color:var(--mu);font-size:13px;}

/* ── Main Content ── */
.main-content{margin-left:240px;padding:22px 22px 60px;min-height:100vh;transition:margin-left .35s;}
.hero{background:linear-gradient(135deg,var(--a) 0%,var(--a2) 100%);border-radius:22px;padding:26px 24px;display:flex;align-items:center;justify-content:space-between;flex-wrap:wrap;gap:16px;margin-bottom:20px;position:relative;overflow:hidden;box-shadow:0 10px 36px rgba(27,67,50,.22);animation:fadeInUp .6s cubic-bezier(.16,1,.3,1) both;}
.hero-badge{display:inline-flex;align-items:center;gap:5px;background:rgba(255,255,255,.18);border:1px solid rgba(255,255,255,.25);border-radius:100px;padding:4px 12px;font-size:11px;font-weight:700;letter-spacing:1.5px;color:#fff;text-transform:uppercase;margin-bottom:10px;}
.hdot{width:5px;height:5px;border-radius:50%;background:#4ade80;animation:pulse 2s infinite;}
.hero-t{font-family:'Montserrat',sans-serif;font-size:clamp(19px,4vw,27px);font-weight:900;color:#fff;line-height:1.2;margin-bottom:6px;}
.hero-s{font-size:13px;color:rgba(255,255,255,.75);line-height:1.6;}
.hero-logo{height:52px;filter:brightness(0) invert(1) opacity(.65);flex-shrink:0;animation:float 4s ease-in-out infinite;}
.stats{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin-bottom:20px;}
.stat{background:var(--glass);backdrop-filter:blur(12px);border:1px solid var(--glass-border);border-radius:16px;padding:16px 14px;transition:all .35s cubic-bezier(.4,0,.2,1);animation:fadeInUp .5s cubic-bezier(.16,1,.3,1) both;}
.stat:nth-child(1){animation-delay:.1s;}.stat:nth-child(2){animation-delay:.15s;}.stat:nth-child(3){animation-delay:.2s;}
.stat:hover{transform:translateY(-4px);box-shadow:var(--sh2);}
.stat-ic{font-size:20px;margin-bottom:7px;}
.stat-v{font-family:'Montserrat',sans-serif;font-size:22px;font-weight:900;color:var(--p);}
.stat-l{font-size:11px;color:var(--mu);margin-top:2px;}
.card{background:var(--glass);backdrop-filter:blur(12px);border:1px solid var(--glass-border);border-radius:18px;padding:20px;margin-bottom:14px;box-shadow:var(--sh-glass);animation:fadeInUp .5s cubic-bezier(.16,1,.3,1) both;}
.card-h{display:flex;align-items:center;justify-content:space-between;margin-bottom:14px;padding-bottom:11px;border-bottom:1px solid var(--glass-border);}
.card-t{font-size:14px;font-weight:700;color:var(--p);}
.list{display:flex;flex-direction:column;gap:7px;}
.notice-item{padding:13px 14px;border-radius:13px;cursor:pointer;transition:all .25s cubic-bezier(.4,0,.2,1);border:1px solid var(--glass-border);background:rgba(240,247,242,.6);}
.notice-item:hover{background:rgba(255,255,255,.8);border-color:var(--bd);transform:translateX(4px);}
.ni-type{display:inline-flex;align-items:center;padding:2px 8px;border-radius:6px;font-size:10px;font-weight:700;letter-spacing:.5px;margin-bottom:5px;}
.nt-n{background:rgba(224,231,255,.7);color:#4338ca;}
.nt-u{background:rgba(237,233,254,.7);color:#7c3aed;}
.ni-title{font-size:14px;font-weight:700;color:var(--p);margin-bottom:2px;}
.ni-meta{font-size:11px;color:var(--mu);}
.ni-body{font-size:13px;color:var(--mu);line-height:1.7;margin-top:8px;display:none;padding-top:8px;border-top:1px solid var(--bd2);}
.notice-item.expanded .ni-body{display:block;animation:fadeInUp .3s both;}
.sch-item{display:flex;gap:12px;padding:13px 14px;border-radius:13px;border:1px solid var(--glass-border);background:rgba(240,247,242,.6);transition:all .25s;}
.sch-item:hover{transform:translateX(4px);background:rgba(255,255,255,.7);}
.sch-date{flex-shrink:0;text-align:center;width:44px;}
.sch-day{font-family:'Montserrat',sans-serif;font-size:22px;font-weight:900;color:var(--p);line-height:1;}
.sch-mon{font-size:10px;color:var(--mu);margin-top:1px;}
.sch-badge{display:inline-flex;align-items:center;padding:2px 8px;border-radius:6px;font-size:10px;font-weight:700;margin-bottom:4px;}
.sb-shoot{background:#fef3c7;color:#92400e;}
.sb-club{background:#ede9fe;color:#7c3aed;}
.sb-meeting{background:#dbeafe;color:#1d4ed8;}
.sch-title{font-size:13px;font-weight:700;color:var(--p);margin-bottom:2px;}
.sch-note{font-size:11px;color:var(--mu);}
.att-row{display:flex;align-items:center;gap:10px;padding:10px 13px;border-radius:12px;border:1px solid var(--glass-border);background:rgba(240,247,242,.6);transition:all .25s;}
.att-row:hover{transform:translateX(4px);}
.att-date{font-size:12px;font-weight:700;color:var(--p);width:84px;flex-shrink:0;}
.att-p{padding:2px 8px;border-radius:6px;font-size:11px;font-weight:600;background:#d1fae5;color:#065f46;}
.att-a{padding:2px 8px;border-radius:6px;font-size:11px;font-weight:600;background:#fee2e2;color:#991b1b;}
.drive-item{display:flex;align-items:center;gap:12px;padding:13px 14px;border-radius:13px;border:1px solid var(--glass-border);background:rgba(240,247,242,.6);cursor:pointer;transition:all .25s;}
.drive-item:hover{background:rgba(255,255,255,.7);border-color:var(--bd);transform:translateX(4px);}
.drive-ic{width:40px;height:40px;border-radius:11px;display:flex;align-items:center;justify-content:center;font-size:20px;background:rgba(216,243,220,.5);flex-shrink:0;}
.drive-name{font-size:13px;font-weight:700;color:var(--p);overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}
.drive-meta{font-size:11px;color:var(--mu);margin-top:2px;}
.info-row{display:flex;align-items:center;gap:12px;padding:11px 0;border-bottom:1px solid var(--bd2);transition:all .2s;}
.info-row:hover{padding-left:6px;}
.info-row:last-child{border-bottom:none;}
.info-k{font-size:12px;font-weight:700;color:var(--mu);width:78px;flex-shrink:0;}
.info-v{font-size:14px;font-weight:600;color:var(--p);}
.tab-c{display:none;}
.tab-c.active{display:block;animation:fadeInUp .4s cubic-bezier(.16,1,.3,1) both;}
.empty{text-align:center;padding:32px;color:var(--mu);font-size:13px;}
.empty-ic{font-size:28px;margin-bottom:7px;opacity:.5;animation:float 3s ease-in-out infinite;}

/* ── DM Chat ── */
.dm-layout{display:grid;grid-template-columns:180px 1fr;gap:12px;min-height:380px;}
.dm-user-list{border:1px solid var(--glass-border);border-radius:14px;background:rgba(240,247,242,.5);overflow:auto;max-height:400px;padding:6px;}
.dm-user-item{padding:9px 10px;border-radius:10px;cursor:pointer;transition:all .2s;font-size:13px;font-weight:600;color:var(--p);}
.dm-user-item:hover{background:rgba(216,243,220,.5);}
.dm-user-item.active{background:var(--lt);color:var(--a);}
.dm-user-role{font-size:10px;color:var(--mu);font-weight:400;}
.dm-chat{display:flex;flex-direction:column;height:400px;}
.dm-messages{flex:1;border:1px solid var(--glass-border);border-radius:14px;padding:12px;background:rgba(240,247,242,.4);overflow-y:auto;margin-bottom:10px;}
.dm-msg{padding:8px 12px;border-radius:12px;margin-bottom:6px;max-width:80%;font-size:13px;line-height:1.5;animation:fadeInUp .2s both;}
.dm-msg-mine{background:rgba(216,243,220,.7);margin-left:auto;border-bottom-right-radius:4px;}
.dm-msg-other{background:rgba(255,255,255,.8);border-bottom-left-radius:4px;}
.dm-msg-meta{font-size:10px;color:var(--mu);margin-bottom:2px;}
.dm-input-row{display:flex;gap:8px;}
.dm-typing{font-size:11px;color:var(--a);padding:4px 0;animation:pulse 1.5s infinite;}

/* ── Now Brief (Samsung style) ── */
.nb-overlay{position:fixed;inset:0;z-index:10000;background:rgba(0,0,0,.5);backdrop-filter:blur(16px);display:none;align-items:flex-start;justify-content:center;padding:20px;overflow-y:auto;}
.nb-overlay.open{display:flex;animation:fadeIn .3s both;}
.nb-container{width:100%;max-width:440px;margin-top:40px;animation:slideUp .5s cubic-bezier(.16,1,.3,1) both;}
@keyframes slideUp{0%{opacity:0;transform:translateY(60px);}100%{opacity:1;transform:translateY(0);}}
.nb-header{text-align:center;margin-bottom:20px;color:#fff;}
.nb-header h2{font-family:'Montserrat',sans-serif;font-size:24px;font-weight:900;margin-bottom:4px;}
.nb-header p{font-size:13px;opacity:.7;}
.nb-card{background:rgba(255,255,255,.9);backdrop-filter:blur(20px);border-radius:18px;padding:18px;margin-bottom:12px;border:1px solid rgba(255,255,255,.5);box-shadow:0 8px 24px rgba(0,0,0,.1);transition:all .3s;}
.nb-card:hover{transform:translateY(-3px) scale(1.01);box-shadow:0 12px 32px rgba(0,0,0,.15);}
.nb-card-icon{font-size:28px;margin-bottom:8px;animation:float 3s ease-in-out infinite;}
.nb-card-title{font-size:15px;font-weight:700;color:var(--p);margin-bottom:4px;}
.nb-card-body{font-size:13px;color:var(--mu);line-height:1.6;}
.nb-card-value{font-family:'Montserrat',sans-serif;font-size:28px;font-weight:900;color:var(--a);margin:8px 0;}
.nb-close{position:fixed;top:20px;right:20px;width:36px;height:36px;border-radius:50%;background:rgba(255,255,255,.2);backdrop-filter:blur(8px);border:none;color:#fff;font-size:18px;cursor:pointer;display:flex;align-items:center;justify-content:center;transition:all .2s;z-index:10001;}
.nb-close:hover{background:rgba(255,255,255,.4);transform:rotate(90deg);}

/* ── Passkey Manager ── */
.pk-item{display:flex;align-items:center;gap:12px;padding:12px 14px;border-radius:13px;border:1px solid var(--glass-border);background:rgba(240,247,242,.5);margin-bottom:6px;animation:fadeInUp .3s both;}
.pk-item:hover{background:rgba(255,255,255,.7);}
.pk-icon{font-size:24px;width:40px;height:40px;display:flex;align-items:center;justify-content:center;background:rgba(216,243,220,.5);border-radius:10px;}
.pk-info{flex:1;}
.pk-name{font-size:13px;font-weight:700;color:var(--p);}
.pk-date{font-size:11px;color:var(--mu);}

/* ── Device Item ── */
.device-item{display:flex;align-items:center;gap:12px;padding:13px 14px;border-radius:13px;border:1px solid var(--glass-border);background:rgba(240,247,242,.5);margin-bottom:6px;transition:all .25s;}
.device-item:hover{transform:translateX(4px);}
.device-icon{font-size:22px;width:40px;height:40px;display:flex;align-items:center;justify-content:center;background:rgba(216,243,220,.5);border-radius:10px;}
.device-info{flex:1;min-width:0;}
.device-name{font-size:13px;font-weight:700;color:var(--p);}
.device-meta{font-size:11px;color:var(--mu);}
.device-current{font-size:10px;color:var(--a);font-weight:700;background:rgba(216,243,220,.7);padding:2px 6px;border-radius:4px;margin-left:4px;}

/* ── Responsive ── */
@media(max-width:768px){
  .sidebar{transform:translateX(-100%);}
  .sidebar.open{transform:translateX(0);box-shadow:0 0 40px rgba(0,0,0,.2);}
  .hamburger{display:flex;}
  .sb-overlay.open{display:block;}
  .topbar{margin-left:0;padding-left:56px;}
  .main-content{margin-left:0;padding:16px 14px 60px;}
  .stats{grid-template-columns:1fr 1fr 1fr;}
  .stat-v{font-size:18px;}
  .notif-panel{width:calc(100vw - 28px);}
  .dm-layout{grid-template-columns:1fr;min-height:auto;}
  .dm-user-list{max-height:150px;}
  .dm-chat{height:300px;}
}
@media(max-width:340px){
  .stats{grid-template-columns:1fr 1fr;}
}
</style></head><body>
<div class="orb o1"></div><div class="orb o2"></div>

<!-- Hamburger Button (Mobile) -->
<button class="hamburger" id="hamburger" onclick="toggleSidebar()">
  <span></span><span></span><span></span>
</button>

<!-- Sidebar Overlay (Mobile) -->
<div class="sb-overlay" id="sb-overlay" onclick="closeSidebar()"></div>

<!-- Sidebar -->
<aside class="sidebar" id="sidebar">
  <div class="sb-logo"><img src="${LOGO_URL}" alt="" onerror="this.style.display='none'"><span class="sb-logo-n">컨텐츠부 포털</span></div>
  <div class="sb-user">
    <div class="sb-avatar">${esc((session.name||"?").charAt(0))}</div>
    <div><div class="sb-uname">${name}</div><div class="sb-urole">${esc(rlbl(rl))}</div></div>
  </div>
  <nav class="sb-nav">
    <div class="sb-section">메뉴</div>
    <button class="sb-item active" onclick="switchTab('notices',this)"><span class="sb-ic">📢</span><span>공지·업데이트</span></button>
    <button class="sb-item" onclick="switchTab('schedules',this)"><span class="sb-ic">📅</span><span>일정</span></button>
    <button class="sb-item" onclick="switchTab('attendance',this)"><span class="sb-ic">📊</span><span>출석</span></button>
    <button class="sb-item" onclick="switchTab('drive',this)"><span class="sb-ic">📁</span><span>드라이브</span></button>
    <div class="sb-section">커뮤니케이션</div>
    <button class="sb-item" onclick="switchTab('dm',this)"><span class="sb-ic">💬</span><span>DM</span><span class="sb-badge" id="dm-badge" style="display:none">0</span></button>
    <div class="sb-section">계정</div>
    <button class="sb-item" onclick="switchTab('devices',this)"><span class="sb-ic">📱</span><span>기기관리</span></button>
    <button class="sb-item" onclick="switchTab('passkeys',this)"><span class="sb-ic">🔑</span><span>패스키 관리</span></button>
    <button class="sb-item" onclick="switchTab('prefs',this)"><span class="sb-ic">⚙️</span><span>설정</span></button>
    <button class="sb-item" onclick="switchTab('myinfo',this)"><span class="sb-ic">👤</span><span>내 정보</span></button>
    ${isA ? '<div class="sb-section">관리</div><a class="sb-item" href="/admin"><span class="sb-ic">🛡️</span><span>관리자 대시보드</span></a>' : ""}
  </nav>
  <div class="sb-foot">
    <button class="sb-logout" onclick="doLogout()">🚪 로그아웃</button>
  </div>
</aside>

<!-- Top Bar -->
<header class="topbar">
  <div style="margin-right:auto;font-size:13px;font-weight:600;color:var(--mu)" id="hero-greet">${nowGreeting()} <strong style="color:var(--a)">${name}</strong>님</div>
  <button class="notif-btn" id="nb" onclick="toggleNotif()">
    <span class="notif-ic">🔔</span>
    <span>알림</span>
    <span class="nbadge" id="nbadge"></span>
  </button>
</header>

<!-- Notification Panel -->
<div class="notif-panel" id="np">
  <div class="np-head">
    <div class="np-t">🔔 알림</div>
    <div style="display:flex;gap:6px">
      <button class="btn bxs bg2" onclick="markRead()">모두 읽음</button>
      <button class="btn bxs bdanger" onclick="clearNotifs()">전체 삭제</button>
    </div>
  </div>
  <div class="np-body" id="np-body"><div class="np-empty">알림이 없습니다</div></div>
</div>

<!-- Main Content Area -->
<main class="main-content">
  ${isA ? '<div id="adm-popup" style="position:fixed;inset:0;z-index:9999;background:rgba(0,0,0,.4);backdrop-filter:blur(12px);display:flex;align-items:center;justify-content:center;padding:16px"><div class="glass-card" style="padding:28px 24px;max-width:340px;width:100%;text-align:center"><div style="font-size:42px;margin-bottom:10px;animation:float 3s ease-in-out infinite">🛡️</div><div style="font-family:Montserrat,sans-serif;font-size:19px;font-weight:900;color:var(--p);margin-bottom:5px">관리자 로그인</div><div style="font-size:13px;color:var(--mu);line-height:1.6;margin-bottom:18px">안녕하세요, <strong>'+name+'</strong>님!<br>관리자 대시보드로 이동하시겠습니까?</div><div style="display:grid;grid-template-columns:1fr 1fr;gap:9px"><button class="btn bp" onclick="location.href=\'/admin\'">🛡️ 대시보드</button><button class="btn bg2" onclick="document.getElementById(\'adm-popup\').style.display=\'none\'">나중에</button></div></div></div>' : ""}

  <!-- Hero -->
  <div class="hero">
    <div>
      <div class="hero-badge"><div class="hdot"></div>CONTENTS DEPT · ${esc(rlbl(rl))}</div>
      <div class="hero-t">산자연중학교<br>컨텐츠부 포털</div>
      <div class="hero-s">${nowGreeting()} <strong style="color:#bbf7d0">${name}</strong>님</div>
      <div class="hero-s" style="margin-top:6px;opacity:.92">공지, 일정, 출석, DM을 한곳에서 확인하세요.</div>
      <div style="margin-top:14px;display:flex;gap:10px;flex-wrap:wrap">
        <button class="btn" style="background:rgba(255,255,255,.2);backdrop-filter:blur(6px);color:#fff;border:1px solid rgba(255,255,255,.3)" onclick="openNowBrief()">✨ 나우 브리프</button>
        <button class="btn" style="background:rgba(255,255,255,.15);backdrop-filter:blur(6px);color:#fff;border:1px solid rgba(255,255,255,.2)" id="notif-perm-btn" onclick="requestBrowserNotifications(true)">🔔 알림 권한 허용</button>
      </div>
    </div>
    <img src="${LOGO_URL}" class="hero-logo" alt="" onerror="this.style.display='none'">
  </div>

  <!-- Stats -->
  <div class="stats">
    <div class="stat"><div class="stat-ic">🎓</div><div class="stat-v">${session.grade||"-"}${session.grade?"학년":""}</div><div class="stat-l">학년</div></div>
    <div class="stat"><div class="stat-ic">🏫</div><div class="stat-v">${session.classNum||"-"}${session.classNum?"반":""}</div><div class="stat-l">반</div></div>
    <div class="stat"><div class="stat-ic">📅</div><div class="stat-v" id="st-sch" style="font-size:16px;margin-top:4px">...</div><div class="stat-l">이번 달 일정</div></div>
  </div>

  <!-- Tab Contents (all properly closed!) -->
  <div class="tab-c active" id="tab-notices">
    <div class="card"><div class="card-h"><div class="card-t">📢 공지사항 & 업데이트</div></div>
      <div class="list" id="notice-list"><div class="empty"><div class="empty-ic">📭</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-schedules">
    <div class="card"><div class="card-h"><div class="card-t">📅 일정표</div>
        <div style="display:flex;gap:5px">
          <button class="btn bxs bg2" onclick="filterSch('all')">전체</button>
          <button class="btn bxs" style="background:#fef3c7;color:#92400e;border:1px solid #fde68a" onclick="filterSch('shoot')">📷</button>
          <button class="btn bxs" style="background:#ede9fe;color:#7c3aed;border:1px solid #ddd6fe" onclick="filterSch('club')">🎯</button>
          <button class="btn bxs" style="background:#dbeafe;color:#1d4ed8;border:1px solid #bfdbfe" onclick="filterSch('meeting')">💼</button>
        </div>
      </div>
      <div class="list" id="sch-list"><div class="empty"><div class="empty-ic">📅</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-attendance">
    <div class="card"><div class="card-h"><div class="card-t">📊 출석 기록</div></div>
      <div class="list" id="att-list"><div class="empty"><div class="empty-ic">📊</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-drive">
    <div class="card"><div class="card-h"><div class="card-t">📁 공유 드라이브</div></div>
      <div class="list" id="drive-list"><div class="empty"><div class="empty-ic">📁</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-dm">
    <div class="card"><div class="card-h"><div class="card-t">💬 실시간 DM</div><span class="dm-typing" id="dm-typing" style="display:none">실시간 연결 중...</span></div>
      <div class="dm-layout">
        <div class="dm-user-list" id="dm-users"></div>
        <div class="dm-chat">
          <div class="dm-messages" id="dm-messages"><div class="np-empty">대화 상대를 선택하세요</div></div>
          <div class="dm-input-row">
            <input class="inp" id="dm-text" placeholder="메시지 입력..." onkeydown="if(event.key==='Enter')sendDm()">
            <button class="btn bp" onclick="sendDm()">전송</button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="tab-c" id="tab-devices">
    <div class="card"><div class="card-h"><div class="card-t">📱 기기 관리</div><div style="display:flex;gap:6px"><button class="btn bsm bg2" onclick="loadMySessions()">새로고침</button><button class="btn bsm bdanger" onclick="logoutAllSessions()">모든 기기 로그아웃</button></div></div>
      <div class="list" id="device-list"><div class="empty"><div class="empty-ic">📱</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-passkeys">
    <div class="card">
      <div class="card-h"><div class="card-t">🔑 패스키 관리</div><button class="btn bsm bp" onclick="registerPasskey()">+ 패스키 등록</button></div>
      <div style="font-size:13px;color:var(--mu);line-height:1.7;margin-bottom:14px;padding:12px;background:rgba(216,243,220,.4);border-radius:12px">
        패스키를 등록하면 비밀번호 없이 <strong>지문, 얼굴 인식, PIN</strong> 등으로 빠르게 로그인할 수 있습니다.
      </div>
      <div class="list" id="passkey-list"><div class="empty"><div class="empty-ic">🔑</div>불러오는 중...</div></div>
    </div>
  </div>

  <div class="tab-c" id="tab-prefs">
    <div class="card"><div class="card-h"><div class="card-t">⚙️ 사용자 설정</div></div>
      <div class="tog-row"><div><div class="tog-l">🔔 브라우저 알림</div><div class="tog-s">공지, 일정, 출석, DM 알림 표시</div></div><label class="toggle"><input type="checkbox" id="pf-push" onchange="savePrefs()"><span class="tsl"></span></label></div>
      <div class="tog-row"><div><div class="tog-l">💬 DM 받기</div><div class="tog-s">다른 사용자의 DM 허용</div></div><label class="toggle"><input type="checkbox" id="pf-dm" onchange="savePrefs()"><span class="tsl"></span></label></div>
      <div class="tog-row"><div><div class="tog-l">🌙 다크모드 (준비중)</div><div class="tog-s">어두운 테마로 전환</div></div><label class="toggle"><input type="checkbox" disabled><span class="tsl" style="opacity:.5"></span></label></div>
    </div>
  </div>

  <div class="tab-c" id="tab-myinfo">
    <div class="card"><div class="card-h"><div class="card-t">👤 내 정보</div><button class="btn bsm bg2" onclick="oModal('m-myinfo')">✏️ 수정</button></div>
      <div class="info-row"><div class="info-k">이름</div><div class="info-v">${name}</div></div>
      <div class="info-row"><div class="info-k">아이디</div><div class="info-v">${esc(session.username)}</div></div>
      <div class="info-row"><div class="info-k">학년/반</div><div class="info-v">${session.grade ? session.grade+"학년 "+session.classNum+"반" : "-"}</div></div>
      <div class="info-row"><div class="info-k">역할</div><div class="info-v">${esc(rlbl(rl))}</div></div>
      <div class="info-row"><div class="info-k">패스키</div><div class="info-v" id="pk-count">0개 등록됨</div></div>
    </div>
  </div>

</main>

<!-- Now Brief Overlay (Samsung Style) -->
<div class="nb-overlay" id="nb-overlay" onclick="if(event.target===this)closeNowBrief()">
  <button class="nb-close" onclick="closeNowBrief()">✕</button>
  <div class="nb-container" id="nb-container"></div>
</div>

<!-- My Info Edit Modal -->
<div class="modal" id="m-myinfo">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-myinfo')">✕</button>
    <div class="m-t">✏️ 내 정보 수정</div>
    <div class="m-s">아이디 또는 비밀번호를 변경할 수 있습니다.</div>
    <div class="err" id="mi-err"></div>
    <div class="suc" id="mi-suc"></div>
    <form autocomplete="on">
    <div class="form-row"><label class="lbl">새 아이디 (변경 시 입력)</label><input class="inp" type="text" id="mi-un" placeholder="비우면 변경 없음" autocomplete="username"></div>
    <div class="form-row"><label class="lbl">새 비밀번호 (변경 시 입력)</label><div class="pw-box"><input class="inp" type="password" id="mi-npw" placeholder="비우면 변경 없음" autocomplete="new-password"><button type="button" class="pw-toggle" onclick="togglePw('mi-npw',this)">보기</button></div></div>
    <div class="form-row"><label class="lbl">현재 비밀번호 * 필수</label><div class="pw-box"><input class="inp" type="password" id="mi-cpw" placeholder="현재 비밀번호 확인" autocomplete="current-password"><button type="button" class="pw-toggle" onclick="togglePw('mi-cpw',this)">보기</button></div></div>
    </form>
    <div style="display:flex;gap:8px">
      <button class="btn bp" style="flex:1" onclick="doMyInfo()">저장</button>
      <button class="btn bg2" onclick="cModal('m-myinfo')">취소</button>
    </div>
  </div>
</div>

<!-- Drive Detail Modal -->
<div class="modal" id="m-drive-detail">
  <div class="mbox"><button class="m-x" onclick="cModal('m-drive-detail')">✕</button><div id="drive-detail-content"></div></div>
</div>

<div class="toast-wrap" id="tw"></div>
${commonJS()}
<script>
var MY_ROLE="${esc(rl)}";
var ROLE_ORDER=["student","member","staff","teacher","bujang","faculty","admin"];
function rlvl(r){return ROLE_ORDER.indexOf(r);}
var schAll=[],drivesAll=[],CURRENT_SID="${esc(session.sid||'')}",dmWith="",USER_NAME="${name}";
var loadedTabs={notices:false,schedules:false,attendance:false,drive:false};
var dmPollInterval=null;

/* ── Sidebar Toggle ── */
function toggleSidebar(){
  var sb=document.getElementById('sidebar');
  var ov=document.getElementById('sb-overlay');
  var hb=document.getElementById('hamburger');
  sb.classList.toggle('open');
  ov.classList.toggle('open');
  hb.classList.toggle('open');
}
function closeSidebar(){
  var sb=document.getElementById('sidebar');
  var ov=document.getElementById('sb-overlay');
  var hb=document.getElementById('hamburger');
  sb.classList.remove('open');
  ov.classList.remove('open');
  hb.classList.remove('open');
}

/* ── Tab Switch ── */
function switchTab(id,btn){
  document.querySelectorAll('.tab-c').forEach(function(x){x.classList.remove('active');});
  document.querySelectorAll('.sb-item').forEach(function(x){x.classList.remove('active');});
  var tab=document.getElementById('tab-'+id);
  if(!tab){toast('탭을 불러오지 못했습니다.','e');return;}
  tab.classList.add('active');
  if(btn)btn.classList.add('active');
  closeSidebar();
  if(id==='notices'&&!loadedTabs.notices)loadNotices();
  if(id==='schedules'&&!loadedTabs.schedules)loadSchedules();
  if(id==='attendance'&&!loadedTabs.attendance)loadAttendance();
  if(id==='drive'&&!loadedTabs.drive)loadDrives();
  if(id==='devices')loadMySessions();
  if(id==='dm'){loadDmUsers(); if(dmWith) loadDms(); startDmPoll();}
  if(id==='prefs')loadPrefs();
  if(id==='passkeys')loadPasskeys();
  if(id==='myinfo')loadMyInfo();
}

/* ── Notifications ── */
var npOpen=false;
function toggleNotif(){
  npOpen=!npOpen;
  document.getElementById('np').classList.toggle('open',npOpen);
  if(npOpen)loadNotifs();
}
document.addEventListener('click',function(ev){
  if(!ev.target.closest('#nb')&&!ev.target.closest('#np')){npOpen=false;document.getElementById('np').classList.remove('open');}
});

/* ── Now Brief (Samsung Style) ── */
function openNowBrief(){
  try{
    var badge=document.getElementById('nbadge');
    var unread=(badge&&badge.style.display!=='none')?(badge.textContent||'0'):'0';
    var noticeCount=document.querySelectorAll('#notice-list .notice-item').length||0;
    var scheduleCount=Array.isArray(schAll)?schAll.length:0;
    var greetEl=document.getElementById('hero-greet');
    var greetText=greetEl?greetEl.textContent:'안녕하세요.';
    var now=new Date();
    var timeStr=now.getHours()+':'+String(now.getMinutes()).padStart(2,'0');

    var c=document.getElementById('nb-container');
    c.innerHTML='';
    c.innerHTML+='<div class="nb-header"><h2>Now Brief</h2><p>'+greetText+'</p></div>';

    var cards=[
      {icon:'🕐',title:'현재 시각',body:'<div class="nb-card-value">'+timeStr+'</div>오늘도 활기찬 하루 되세요!',delay:'.1s'},
      {icon:'📢',title:'공지사항',body:'<div class="nb-card-value">'+noticeCount+'<span style="font-size:14px;color:var(--mu)"> 건</span></div>등록된 공지·업데이트',delay:'.2s'},
      {icon:'📅',title:'예정된 일정',body:'<div class="nb-card-value">'+scheduleCount+'<span style="font-size:14px;color:var(--mu)"> 건</span></div>확인이 필요한 일정',delay:'.3s'},
      {icon:'🔔',title:'미확인 알림',body:'<div class="nb-card-value">'+unread+'<span style="font-size:14px;color:var(--mu)"> 건</span></div>지금 바로 확인하세요.',delay:'.4s'},
      {icon:'💡',title:'팁',body:'사이드바에서 모든 기능에 빠르게 접근하세요.<br>패스키를 등록하면 더 빠르게 로그인할 수 있습니다!',delay:'.5s'}
    ];
    cards.forEach(function(cd){
      var d=document.createElement('div');
      d.className='nb-card';
      d.style.animation='slideUp .5s '+cd.delay+' cubic-bezier(.16,1,.3,1) both';
      d.innerHTML='<div class="nb-card-icon">'+cd.icon+'</div><div class="nb-card-title">'+cd.title+'</div><div class="nb-card-body">'+cd.body+'</div>';
      c.appendChild(d);
    });

    document.getElementById('nb-overlay').classList.add('open');
  }catch(ex){toast('나우 브리프를 열지 못했습니다.','e');}
}
function closeNowBrief(){
  document.getElementById('nb-overlay').classList.remove('open');
}

/* ── Device Sessions ── */
async function loadMySessions(){
  try{
    var r=await fetch('/api/my-sessions'); var d=await r.json();
    var el=document.getElementById('device-list'); if(!el) return;
    el.innerHTML='';
    var rows=d.sessions||[];
    if(!rows.length){el.innerHTML='<div class="empty"><div class="empty-ic">📱</div>로그인된 기기가 없습니다.</div>';return;}
    rows.forEach(function(s,idx){
      var item=document.createElement('div'); item.className='device-item';
      item.style.animationDelay=(idx*0.05)+'s';
      var isMe=CURRENT_SID===s.sid;
      var iconText=s.device&&s.device.indexOf('Android')>=0?'📱':s.device&&s.device.indexOf('iOS')>=0?'📱':s.device&&s.device.indexOf('Windows')>=0?'💻':'🖥️';
      item.innerHTML='<div class="device-icon">'+iconText+'</div><div class="device-info"><div class="device-name">'+e(s.device)+(isMe?'<span class="device-current">현재 기기</span>':'')+'</div><div class="device-meta">IP '+e(s.ip)+' · '+e(s.loginAt)+'</div></div>';
      var btn=document.createElement('button');
      btn.className='btn bsm '+(isMe?'bdanger':'bg2');
      btn.textContent='로그아웃';
      btn.onclick=function(){logoutSession(s.sid,isMe);};
      item.appendChild(btn);
      el.appendChild(item);
    });
  }catch(ex){toast('기기 목록을 불러오지 못했습니다.','e');}
}
async function logoutSession(sid,self){
  try{ var r=await fetch('/api/logout-session',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({sid:sid})}); var d=await r.json();
    if(d.success){ if(self||d.self) location.href='/'; else {toast('기기에서 로그아웃했습니다.','s'); loadMySessions();} }
  }catch(ex){toast('로그아웃 실패','e');}
}
async function logoutAllSessions(){
  sjyConfirm('모든 기기에서 로그아웃하시겠습니까?',function(){
    fetch('/api/logout-all',{method:'POST'}).then(function(r){return r.json();}).then(function(d){if(d.success) location.href='/';});
  },{okText:'전체 로그아웃',cancelText:'취소'});
}

/* ── Passkey Management ── */
async function loadPasskeys(){
  try{
    var r=await fetch('/api/passkey/list'); var d=await r.json();
    var el=document.getElementById('passkey-list'); if(!el) return;
    el.innerHTML='';
    var pks=d.passkeys||[];
    var cnt=document.getElementById('pk-count');
    if(cnt) cnt.textContent=pks.length+'개 등록됨';
    if(!pks.length){el.innerHTML='<div class="empty"><div class="empty-ic">🔑</div>등록된 패스키가 없습니다.<br>위 버튼으로 첫 패스키를 등록해보세요!</div>';return;}
    pks.forEach(function(pk,idx){
      var item=document.createElement('div'); item.className='pk-item';
      item.style.animationDelay=(idx*0.05)+'s';
      item.innerHTML='<div class="pk-icon">🔑</div><div class="pk-info"><div class="pk-name">'+e(pk.name||'패스키')+'</div><div class="pk-date">'+e(pk.createdAt||'')+'</div></div>';
      var btn=document.createElement('button');
      btn.className='btn bsm bdanger';
      btn.textContent='삭제';
      btn.onclick=function(){deletePasskey(pk.id);};
      item.appendChild(btn);
      el.appendChild(item);
    });
  }catch(ex){toast('패스키 목록 로드 실패','e');}
}
async function deletePasskey(id){
  sjyConfirm('이 패스키를 삭제하시겠습니까?',function(){
    fetch('/api/passkey/delete',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({passkeyId:id})}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast('패스키가 삭제되었습니다.','s');loadPasskeys();}else toast(d.message||'삭제 실패','e');
    });
  },{danger:true,okText:'삭제',cancelText:'취소'});
}
async function registerPasskey(){
  if(!window.PublicKeyCredential){toast('이 브라우저는 패스키를 지원하지 않습니다.','e');return;}
  try{
    var r=await fetch('/api/passkey/register-options',{method:'POST'}); var d=await r.json();
    if(!d.success){toast(d.message||'패스키 등록 준비 실패','e');return;}
    var pk=d.publicKey;
    pk.challenge=b64uToArr(pk.challenge);
    pk.user.id=b64uToArr(pk.user.id);
    pk.excludeCredentials=(pk.excludeCredentials||[]).map(function(x){return {id:b64uToArr(x.id),type:x.type};});
    var cred=await navigator.credentials.create({publicKey:pk});
    var publicKeyBuf=cred.response.getPublicKey?cred.response.getPublicKey():null;
    var body={challengeId:d.challengeId,id:cred.id,name:'패스키 ('+new Date().toLocaleDateString('ko-KR')+')',response:{clientDataJSON:arrToB64u(cred.response.clientDataJSON),publicKey:publicKeyBuf?arrToB64u(publicKeyBuf):''}};
    var r2=await fetch('/api/passkey/register-complete',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)});
    var d2=await r2.json();
    if(d2.success){toast('패스키가 등록되었습니다! 🎉','s');loadPasskeys();}else toast(d2.message||'패스키 등록 실패','e');
  }catch(ex){toast('패스키 등록에 실패했습니다.','e');}
}

/* ── Prefs ── */
async function loadPrefs(){
  try{ var r=await fetch('/api/me',{cache:'no-store'}); var d=await r.json(); if(d.success&&d.prefs){ var p=document.getElementById('pf-push'); var m=document.getElementById('pf-dm'); if(p) p.checked=!!d.prefs.pushEnabled; if(m) m.checked=!!d.prefs.dmEnabled; } }catch(ex){}
}
async function savePrefs(){
  try{
    var body={pushEnabled:document.getElementById('pf-push').checked,dmEnabled:document.getElementById('pf-dm').checked,dmBlocked:!document.getElementById('pf-dm').checked};
    var r=await fetch('/api/save-prefs',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)}); var d=await r.json();
    if(d.success){ if(body.pushEnabled) await requestBrowserNotifications(true); toast('설정이 저장되었습니다.','s'); }
  }catch(ex){toast('설정 저장 실패','e');}
}

/* ── DM (Real-time via fast polling) ── */
async function loadDmUsers(){
  try{ var r=await fetch('/api/dm-users'); var d=await r.json(); var el=document.getElementById('dm-users'); if(!el) return; el.innerHTML='';
    var rows=d.users||[]; if(!rows.length){el.innerHTML='<div class="np-empty" style="padding:20px">대화할 사용자가 없습니다.</div>';return;}
    rows.forEach(function(u){
      var item=document.createElement('div'); item.className='dm-user-item'+(dmWith===u.id?' active':'');
      item.innerHTML=e(u.name)+'<div class="dm-user-role">'+e(u.role)+'</div>';
      item.onclick=function(){dmWith=u.id; el.querySelectorAll('.dm-user-item').forEach(function(x){x.classList.remove('active');}); item.classList.add('active'); loadDms(); startDmPoll();};
      el.appendChild(item);
    });
  }catch(ex){toast('DM 사용자 목록을 불러오지 못했습니다.','e');}
}
async function loadDms(){
  if(!dmWith) return;
  try{
    var r=await fetch('/api/dms?with='+encodeURIComponent(dmWith)); var d=await r.json();
    var el=document.getElementById('dm-messages'); if(!el) return;
    var wasAtBottom=el.scrollHeight-el.scrollTop-el.clientHeight<50;
    el.innerHTML='';
    if(!(d.messages||[]).length){el.innerHTML='<div class="np-empty" style="padding:40px">메시지가 없습니다.<br>첫 메시지를 보내보세요! 💬</div>';return;}
    (d.messages||[]).forEach(function(m){
      var row=document.createElement('div');
      var isMine=m.fromName===USER_NAME;
      row.className='dm-msg '+(isMine?'dm-msg-mine':'dm-msg-other');
      row.innerHTML='<div class="dm-msg-meta">'+e(m.fromName)+' · '+e(m.at)+'</div>'+e(m.text);
      el.appendChild(row);
    });
    if(wasAtBottom) el.scrollTop=el.scrollHeight;
  }catch(ex){}
}
function startDmPoll(){
  if(dmPollInterval) clearInterval(dmPollInterval);
  dmPollInterval=setInterval(function(){if(dmWith)loadDms();},2000);
  var typing=document.getElementById('dm-typing');
  if(typing){typing.style.display='inline';setTimeout(function(){typing.textContent='실시간 연결됨 ✓';typing.style.animation='none';},1500);}
}
async function sendDm(){
  var t=document.getElementById('dm-text'); if(!dmWith){toast('대화 상대를 먼저 선택하세요.','i');return;} if(!t.value.trim()){toast('메시지를 입력하세요.','i');return;}
  try{ var r=await fetch('/api/dms',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({toId:dmWith,text:t.value})}); var d=await r.json(); if(d.success){t.value=''; loadDms();} else toast(d.message||'전송 실패','e'); }catch(ex){toast('전송 실패','e');}
}

/* ── Load Functions ── */
async function loadNotifs(){
  try{
    var r=await fetch('/api/notifs');var d=await r.json();
    if(!d.success)return;
    var badge=document.getElementById('nbadge');
    if(d.unread>0){badge.textContent=d.unread>9?'9+':String(d.unread);badge.style.display='flex';}
    else{badge.style.display='none';}
    var body=document.getElementById('np-body');
    if(!d.notifs||!d.notifs.length){body.innerHTML='<div class="np-empty">알림이 없습니다</div>';return;}
    body.innerHTML='';
    d.notifs.forEach(function(n){
      var item=document.createElement('div');
      item.className='nitem'+(n.read?'':' unread');
      var msg=document.createElement('div');msg.className='nitem-msg';msg.textContent=n.msg;
      var tm=document.createElement('div');tm.className='nitem-time';tm.textContent=n.at;
      item.appendChild(msg);item.appendChild(tm);body.appendChild(item);
    });
  }catch(ex){}
}
async function markRead(){await fetch('/api/notif-read',{method:'POST'});loadNotifs();toast('모든 알림을 읽음 처리했습니다.','s');}
async function clearNotifs(){await fetch('/api/notif-clear',{method:'POST'});loadNotifs();toast('알림이 삭제되었습니다.','s');}
async function loadNotices(){
  loadedTabs.notices=true;
  try{
    var r=await fetch('/api/notices');var d=await r.json();
    var el=document.getElementById('notice-list');
    if(!d.notices||!d.notices.length){el.innerHTML='<div class="empty"><div class="empty-ic">📭</div>등록된 공지가 없습니다.</div>';return;}
    el.innerHTML='';
    d.notices.forEach(function(n,idx){
      var item=document.createElement('div');
      item.className='notice-item';
      item.style.animationDelay=(idx*0.05)+'s';
      item.style.animation='fadeInUp .4s '+(idx*0.05)+'s cubic-bezier(.16,1,.3,1) both';
      item.onclick=function(){item.classList.toggle('expanded');};
      var badge=document.createElement('span');
      badge.className='ni-type '+(n.noticeType==='update'?'nt-u':'nt-n');
      badge.textContent=n.noticeType==='update'?'🆕 UPDATE':'📢 NOTICE';
      var title=document.createElement('div');title.className='ni-title';title.textContent=n.title;
      var meta=document.createElement('div');meta.className='ni-meta';meta.textContent=n.author+' · '+n.createdAt;
      var body=document.createElement('div');body.className='ni-body';body.textContent=n.content;
      item.appendChild(badge);item.appendChild(title);item.appendChild(meta);item.appendChild(body);
      el.appendChild(item);
    });
  }catch(ex){}
}
async function loadSchedules(){
  loadedTabs.schedules=true;
  try{
    var r=await fetch('/api/schedules');var d=await r.json();schAll=d.schedules||[];renderSchs(schAll);
    var now=new Date(),ym=now.getFullYear()+'-'+String(now.getMonth()+1).padStart(2,'0');
    var cnt=schAll.filter(function(s){return s.date&&s.date.indexOf(ym)===0;}).length;
    document.getElementById('st-sch').textContent=String(cnt)+'건';
  }catch(ex){}
}
function filterSch(t){renderSchs(t==='all'?schAll:schAll.filter(function(s){return s.type===t;}));}
function renderSchs(list){
  var el=document.getElementById('sch-list');el.innerHTML='';
  if(!list||!list.length){el.innerHTML='<div class="empty"><div class="empty-ic">📅</div>등록된 일정이 없습니다.</div>';return;}
  list.forEach(function(s,idx){
    var dt=s.date?new Date(s.date+' 00:00'):null;
    var day=dt?String(dt.getDate()):'-';var mon=dt?String(dt.getMonth()+1)+'월':'';
    var bc={shoot:'sb-shoot',club:'sb-club',meeting:'sb-meeting'}[s.type]||'';
    var bl={shoot:'📷 촬영',club:'🎯 동아리',meeting:'💼 회의'}[s.type]||'📌';
    var item=document.createElement('div');item.className='sch-item';
    item.style.animation='fadeInUp .4s '+(idx*0.05)+'s cubic-bezier(.16,1,.3,1) both';
    var dateDiv=document.createElement('div');dateDiv.className='sch-date';
    dateDiv.innerHTML='<div class="sch-day">'+day+'</div><div class="sch-mon">'+mon+'</div>';
    var info=document.createElement('div');
    info.innerHTML='<span class="sch-badge '+bc+'">'+bl+'</span><div class="sch-title">'+e(s.title)+(s.time?' · '+e(s.time):'')+'</div>'+(s.note?'<div class="sch-note">'+e(s.note)+'</div>':'');
    item.appendChild(dateDiv);item.appendChild(info);el.appendChild(item);
  });
}
async function loadAttendance(){
  loadedTabs.attendance=true;
  try{
    var r=await fetch('/api/attendance');var d=await r.json();
    var el=document.getElementById('att-list');el.innerHTML='';
    if(!d.records||!d.records.length){el.innerHTML='<div class="empty"><div class="empty-ic">📊</div>출석 기록이 없습니다.</div>';return;}
    d.records.forEach(function(rec,idx){
      var row=document.createElement('div');row.className='att-row';
      row.style.animation='fadeInUp .4s '+(idx*0.05)+'s cubic-bezier(.16,1,.3,1) both';
      var dt=document.createElement('div');dt.className='att-date';dt.textContent=rec.date;
      var tags=document.createElement('div');tags.style.cssText='display:flex;gap:5px;flex-wrap:wrap';
      if(rec.attendees&&rec.attendees.length){var p=document.createElement('span');p.className='att-p';p.textContent='출석 '+rec.attendees.length+'명';tags.appendChild(p);}
      if(rec.absentees&&rec.absentees.length){var a=document.createElement('span');a.className='att-a';a.textContent='결석 '+rec.absentees.length+'명';tags.appendChild(a);}
      row.appendChild(dt);row.appendChild(tags);
      el.appendChild(row);
    });
  }catch(ex){}
}
async function loadDrives(){
  loadedTabs.drive=true;
  try{
    var r=await fetch('/api/drives');var d=await r.json();drivesAll=d.drives||[];
    var el=document.getElementById('drive-list');el.innerHTML='';
    if(!drivesAll.length){el.innerHTML='<div class="empty"><div class="empty-ic">📁</div>등록된 파일이 없습니다.</div>';return;}
    drivesAll.forEach(function(f,idx){
      var ok=rlvl(MY_ROLE)>=rlvl(f.minRole||'student');
      var exts={pdf:'📄',pptx:'📊',ppt:'📊',docx:'📝',doc:'📝',xlsx:'📈',xls:'📈',zip:'📦',hwp:'📋'};
      var ext=f.filename?f.filename.split('.').pop().toLowerCase():'';
      var ic=exts[ext]||'📎';
      var item=document.createElement('div');item.className='drive-item';
      item.style.animation='fadeInUp .4s '+(idx*0.05)+'s cubic-bezier(.16,1,.3,1) both';
      item.onclick=function(){openDrive(f.id);};
      item.innerHTML='<div class="drive-ic">'+ic+'</div><div style="flex:1;min-width:0"><div class="drive-name">'+e(f.filename)+'</div><div class="drive-meta">'+e(f.category)+' · '+e(f.uploadedBy)+(f.filesize?' · '+e(f.filesize):'')+'</div></div><span style="font-size:14px">'+(ok?'⬇️':'🔒')+'</span>';
      el.appendChild(item);
    });
  }catch(ex){}
}
function openDrive(id){
  var f=drivesAll.find(function(x){return x.id===id;});if(!f)return;
  var ok=rlvl(MY_ROLE)>=rlvl(f.minRole||'student');
  var RLBL2={student:'🎓 학생',member:'👤 멤버',staff:'📸 부서원',teacher:'👩‍🏫 담당선생님',bujang:'👑 부장',faculty:'🏫 교직원',admin:'🛡️ 관리자'};
  var det=document.getElementById('drive-detail-content');det.innerHTML='';
  if(!ok){
    det.innerHTML='<div style="text-align:center;padding:20px"><div style="font-size:44px;margin-bottom:12px;animation:float 3s ease-in-out infinite">🔒</div><div style="font-size:16px;font-weight:700;color:#dc2626;margin-bottom:8px">접근 불가</div><div style="font-size:13px;color:var(--mu)">이 문서는 '+(RLBL2[f.minRole]||f.minRole)+' 이상부터 접근 가능합니다.</div></div>';
  }else{
    det.innerHTML='<div class="m-t">📁 '+e(f.filename)+'</div><div class="ipre" style="margin:12px 0"><div class="ir"><div class="ik">카테고리</div><div class="iv">'+e(f.category)+'</div></div><div class="ir"><div class="ik">업로드</div><div class="iv">'+e(f.uploadedBy)+' · '+e(f.uploadedAt)+'</div></div>'+(f.description?'<div class="ir"><div class="ik">설명</div><div class="iv">'+e(f.description)+'</div></div>':'')+(f.filesize?'<div class="ir"><div class="ik">파일크기</div><div class="iv">'+e(f.filesize)+'</div></div>':'')+'</div><a href="'+e(f.url)+'" target="_blank" class="btn bp" style="width:100%">⬇️ 다운로드 / 열기</a>';
  }
  oModal('m-drive-detail');
}

/* ── My Info ── */
async function loadMyInfo(){
  try{var r=await fetch('/api/me',{cache:'no-store'});var d=await r.json();if(d.success&&d.user){
    var cnt=document.getElementById('pk-count');if(cnt)cnt.textContent=(d.user.passkeys||0)+'개 등록됨';
  }}catch(ex){}
}
async function doMyInfo(){
  var un=document.getElementById('mi-un').value.trim(),npw=document.getElementById('mi-npw').value,cpw=document.getElementById('mi-cpw').value;
  var err=document.getElementById('mi-err'),suc=document.getElementById('mi-suc');
  err.style.display='none';suc.style.display='none';
  if(!cpw){err.textContent='현재 비밀번호를 입력하세요.';err.style.display='block';return;}
  try{
    var r=await fetch('/api/update-myinfo',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({currentPw:cpw,newPw:npw||'',username:un||''})});
    var d=await r.json();
    if(d.success){suc.textContent='✅ 수정 완료! 다시 로그인하시면 반영됩니다.';suc.style.display='block';}
    else{err.textContent=d.message||'수정 실패';err.style.display='block';}
  }catch(ex){err.textContent='서버 오류';err.style.display='block';}
}

/* ── Push Notifications ── */
async function requestBrowserNotifications(force){
  try{
    var btn=document.getElementById('notif-perm-btn');
    if(!('Notification' in window)){if(btn) btn.style.display='none';return false;}
    if(Notification.permission==='granted'){if(btn) btn.style.display='none';return true;}
    if(Notification.permission==='denied'){if(force) toast('브라우저 설정에서 알림 권한을 허용해주세요.','i');return false;}
    if(force!==true) return false;
    var perm=await Notification.requestPermission();
    if(perm==='granted'){if(btn) btn.style.display='none';toast('브라우저 알림이 활성화되었습니다! 🔔','s');return true;}
    if(perm==='denied') toast('브라우저 알림이 차단되었습니다.','i');
    return perm==='granted';
  }catch(ex){ return false; }
}

/* ── Poll / Init ── */
async function pollMe(){
  try{
    var r=await fetch('/api/me',{cache:'no-store'});
    if(r.status===401){ location.href='/'; return; }
    var d=await r.json(); if(!d.success) return;
    if(d.access==='block'||d.access==='maint'||(d.user&&d.user.blocked)){ location.href='/'; return; }
    if(d.user&&d.user.role&&d.user.role!==MY_ROLE){ location.reload(); return; }
  }catch(ex){}
}

loadPrefs();loadMySessions();loadDmUsers();loadNotices();loadNotifs();loadPasskeys();loadMyInfo();
setTimeout(function(){requestBrowserNotifications(true);},1200);
setInterval(loadNotifs,30000);
setInterval(pollMe,5000);
requestBrowserNotifications(false);
document.addEventListener('visibilitychange',function(){ if(!document.hidden){pollMe();loadNotifs();} });
</script></body></html>`;
}


function pageAdmin(session){
  const name = esc(session.name);
  return GHEAD("관리자 대시보드")+`
body{display:flex;min-height:100vh;overflow-x:hidden;background:var(--bg);}
.sb{width:210px;min-height:100vh;background:#fff;border-right:1.5px solid var(--bd);display:flex;flex-direction:column;position:fixed;top:0;left:0;bottom:0;z-index:50;}
.sb-logo{display:flex;align-items:center;gap:8px;padding:16px 14px;border-bottom:1.5px solid var(--bd);}
.sb-logo img{height:26px;}
.sb-logo-n{font-size:13px;font-weight:800;color:var(--p);}
.sb-u{display:flex;align-items:center;gap:8px;padding:11px 13px;margin:8px;background:var(--lt);border:1.5px solid var(--bd);border-radius:13px;}
.sb-av{width:32px;height:32px;border-radius:50%;background:linear-gradient(135deg,var(--a),var(--a2));display:flex;align-items:center;justify-content:center;font-size:13px;font-weight:700;color:#fff;flex-shrink:0;}
.sb-un{font-size:12px;font-weight:700;color:var(--p);}
.sb-ur{font-size:10px;color:var(--a);font-weight:700;letter-spacing:.8px;}
.sb-nav{flex:1;padding:4px 6px;overflow-y:auto;}
.sb-sec{font-size:10px;font-weight:700;letter-spacing:2px;color:#a8c5b0;text-transform:uppercase;padding:10px 10px 4px;}
.sbi{display:flex;align-items:center;gap:7px;padding:9px 11px;border-radius:11px;font-size:13px;font-weight:500;color:var(--mu);cursor:pointer;transition:all .18s;text-decoration:none;margin-bottom:1px;border:none;background:none;width:100%;text-align:left;}
.sbi:hover{background:var(--lt);color:var(--p);}
.sbi.active{background:linear-gradient(135deg,rgba(45,106,79,.12),rgba(27,67,50,.06));color:var(--a);font-weight:700;}
.sbi-ic{font-size:15px;width:19px;text-align:center;}
.sb-badge{background:var(--a);color:#fff;border-radius:100px;font-size:10px;font-weight:700;padding:2px 6px;margin-left:auto;}
.sb-foot{padding:11px 13px;border-top:1.5px solid var(--bd);}
.sb-out{display:flex;align-items:center;gap:7px;font-size:12px;color:var(--mu);text-decoration:none;padding:7px 10px;border-radius:10px;transition:all .18s;border:none;background:none;width:100%;cursor:pointer;}
.sb-out:hover{background:#fff5f5;color:#dc2626;}
.main{margin-left:210px;flex:1;padding:26px 24px;min-height:100vh;}
.tab-c{display:none;}
.tab-c.active{display:block;}
.p-t{font-family:'Montserrat',sans-serif;font-size:21px;font-weight:900;color:var(--p);margin-bottom:3px;}
.p-s{font-size:13px;color:var(--mu);margin-bottom:20px;}
.srow{display:grid;grid-template-columns:repeat(4,1fr);gap:10px;margin-bottom:18px;}
.sc{background:#fff;border:1.5px solid var(--bd);border-radius:15px;padding:16px 18px;}
.sc-ic{font-size:18px;margin-bottom:6px;}
.sc-n{font-family:'Montserrat',sans-serif;font-size:26px;font-weight:900;color:var(--p);}
.sc-l{font-size:11px;color:var(--mu);margin-top:1px;}
.card{background:#fff;border:1.5px solid var(--bd);border-radius:17px;padding:20px;margin-bottom:14px;box-shadow:var(--sh);}
.card-h{display:flex;align-items:center;justify-content:space-between;margin-bottom:14px;padding-bottom:11px;border-bottom:1.5px solid var(--bd2);}
.card-t{font-size:14px;font-weight:700;color:var(--p);}
.list{display:flex;flex-direction:column;gap:7px;}
.li{display:flex;align-items:flex-start;gap:9px;padding:11px 13px;background:var(--bg);border:1.5px solid var(--bd2);border-radius:12px;transition:all .18s;}
.li:hover{background:#fff;border-color:var(--bd);}
.li-b{flex:1;min-width:0;}
.li-t{font-size:13px;font-weight:700;color:var(--p);margin-bottom:2px;line-height:1.35;}
.li-m{font-size:11px;color:var(--mu);line-height:1.4;}
.li-a{display:flex;gap:5px;flex-shrink:0;flex-wrap:wrap;align-items:flex-start;}
.empty{text-align:center;padding:32px;color:var(--mu);font-size:13px;}
.empty-ic{font-size:28px;margin-bottom:7px;opacity:.5;}
@media(max-width:800px){
  .srow{grid-template-columns:1fr 1fr;}
  .sb{width:100%;min-height:auto;position:fixed;bottom:0;top:auto;border-right:none;border-top:1.5px solid var(--bd);flex-direction:row;height:54px;z-index:100;background:rgba(255,255,255,.95);backdrop-filter:blur(14px);}
  .sb-logo,.sb-u,.sb-foot,.sb-sec{display:none;}
  .sb-nav{flex:1;padding:0;display:flex;align-items:center;justify-content:space-around;overflow:visible;}
  .sbi{flex-direction:column;gap:1px;font-size:10px;padding:5px 4px;border-radius:0;margin:0;flex:1;justify-content:center;text-align:center;}
  .sbi-ic{font-size:18px;width:auto;}
  .sb-badge{display:none;}
  .main{margin-left:0;padding:14px 12px 68px;}
}
@media(max-width:500px){.srow{grid-template-columns:1fr 1fr;}}
</style></head><body>
<div class="sb">
  <div class="sb-logo"><img src="${LOGO_URL}" alt="" onerror="this.style.display='none'"><span class="sb-logo-n">컨텐츠부 관리</span></div>
  <div class="sb-u"><div class="sb-av">${esc((session.name||"A").charAt(0))}</div><div><div class="sb-un">${name}</div><div class="sb-ur">🛡️ ADMIN</div></div></div>
  <nav class="sb-nav">
    <div class="sb-sec">메뉴</div>
    <button class="sbi active" onclick="showTab('dash',this)"><span class="sbi-ic">📊</span><span>대시보드</span></button>
    <button class="sbi" onclick="showTab('pend',this)"><span class="sbi-ic">📋</span><span>신청</span><span class="sb-badge" id="pb">0</span></button>
    <button class="sbi" onclick="showTab('mems',this)"><span class="sbi-ic">👥</span><span>멤버</span></button>
    <button class="sbi" onclick="showTab('scheds',this)"><span class="sbi-ic">📅</span><span>일정</span></button>
    <button class="sbi" onclick="showTab('notices',this)"><span class="sbi-ic">📢</span><span>공지</span></button>
    <button class="sbi" onclick="showTab('att',this)"><span class="sbi-ic">📊</span><span>출석</span></button>
    <button class="sbi" onclick="showTab('drive',this)"><span class="sbi-ic">📁</span><span>드라이브</span></button>
    <button class="sbi" onclick="showTab('settings',this)"><span class="sbi-ic">⚙️</span><span>설정</span></button>
    <a href="/main" class="sbi"><span class="sbi-ic">🏠</span><span>홈</span></a>
  </nav>
  <div class="sb-foot"><button class="sb-out" onclick="doLogout()"><span>🚪</span><span>로그아웃</span></button></div>
</div>
<div class="main">

<div class="tab-c active" id="tab-dash">
  <div class="p-t">대시보드</div><div class="p-s">산자연중학교 컨텐츠부 관리자 패널</div>
  <div class="srow">
    <div class="sc"><div class="sc-ic">📋</div><div class="sc-n" id="d-pend">-</div><div class="sc-l">가입 대기</div></div>
    <div class="sc"><div class="sc-ic">👥</div><div class="sc-n" id="d-mems">-</div><div class="sc-l">전체 멤버</div></div>
    <div class="sc"><div class="sc-ic">📅</div><div class="sc-n" id="d-schs">-</div><div class="sc-l">등록 일정</div></div>
    <div class="sc"><div class="sc-ic">📢</div><div class="sc-n" id="d-noti">-</div><div class="sc-l">공지사항</div></div>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">⚡ 빠른 작업</div></div>
    <div style="display:flex;gap:8px;flex-wrap:wrap">
      <button class="btn bp bsm" onclick="showTab('pend',null)">📋 신청 관리</button>
      <button class="btn bg2 bsm" onclick="showTab('scheds',null)">📅 일정 등록</button>
      <button class="btn bg2 bsm" onclick="showTab('notices',null)">📢 공지 작성</button>
      <button class="btn bg2 bsm" onclick="showTab('settings',null)">⚙️ 서버 설정</button>
    </div>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📋 최근 가입 신청</div><button class="btn bp bsm" onclick="showTab('pend',null)">전체 보기</button></div>
    <div class="list" id="d-pend-list"><div class="empty"><div class="empty-ic">📭</div>불러오는 중...</div></div>
  </div>
</div>

<div class="tab-c" id="tab-pend">
  <div class="p-t">가입 신청 관리</div><div class="p-s">대기 중인 신청을 검토하세요.</div>
  <div class="card"><div class="card-h"><div class="card-t">📋 신청 목록</div><span id="pend-cnt" style="font-size:12px;color:var(--mu)">0건</span></div>
    <div class="list" id="pend-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-mems">
  <div class="p-t">멤버 관리</div><div class="p-s">승인된 전체 멤버를 관리합니다.</div>
  <div class="card"><div class="card-h"><div class="card-t">👥 멤버 목록</div><span id="mems-cnt" style="font-size:12px;color:var(--mu)">0명</span></div>
    <input class="inp" type="text" id="mem-srch" placeholder="🔍 이름/아이디 검색" oninput="filterMems(this.value)" style="max-width:280px;margin-bottom:12px">
    <div class="list" id="mems-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-scheds">
  <div class="p-t">일정 관리</div><div class="p-s">촬영, 동아리, 회의 일정을 등록하세요.</div>
  <div class="card"><div class="card-h"><div class="card-t">➕ 일정 등록</div></div>
    <div class="form-row"><label class="lbl">유형 *</label>
      <select class="inp sel" id="sch-type" style="max-width:200px">
        <option value="shoot">📷 촬영일정</option><option value="club">🎯 동아리일정</option><option value="meeting">💼 회의일정</option>
      </select>
    </div>
    <div class="form-row"><label class="lbl">제목 *</label><input class="inp" type="text" id="sch-title" placeholder="일정 제목"></div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px">
      <div class="form-row"><label class="lbl">날짜 *</label><input class="inp" type="date" id="sch-date"></div>
      <div class="form-row"><label class="lbl">시간</label><input class="inp" type="time" id="sch-time"></div>
    </div>
    <div class="form-row"><label class="lbl">메모</label><textarea class="inp" id="sch-note" rows="2" placeholder="일정 설명 (선택)"></textarea></div>
    <div class="form-row"><label class="lbl">촬영 담당자 (알림 전송)</label>
      <div id="assignee-box" style="padding:8px;background:var(--bg);border:1.5px solid var(--bd);border-radius:12px;max-height:160px;overflow-y:auto;display:flex;flex-direction:column;gap:2px"></div>
    </div>
    <button class="btn bp" onclick="addSchedule()">📅 일정 등록</button>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📅 등록된 일정</div></div>
    <div class="list" id="sch-admin-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-notices">
  <div class="p-t">공지사항 &amp; 업데이트</div><div class="p-s">공지사항과 업데이트 소식을 작성하세요.</div>
  <div class="card"><div class="card-h"><div class="card-t">✏️ 새 게시물 작성</div></div>
    <div class="form-row"><label class="lbl">유형</label>
      <select class="inp sel" id="noti-type" style="max-width:200px">
        <option value="notice">📢 공지사항</option><option value="update">🆕 업데이트 소식</option>
      </select>
    </div>
    <div class="form-row"><label class="lbl">제목 *</label><input class="inp" type="text" id="noti-title" placeholder="제목 입력"></div>
    <div class="form-row"><label class="lbl">내용 *</label><textarea class="inp" id="noti-content" rows="5" placeholder="내용 입력"></textarea></div>
    <button class="btn bp" onclick="addNotice()">📢 게시하기 (전체 알림 발송)</button>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📋 게시물 목록</div></div>
    <div class="list" id="notice-admin-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-att">
  <div class="p-t">출석 관리</div><div class="p-s">출석 기록을 등록하고 관리합니다.</div>
  <div class="card"><div class="card-h"><div class="card-t">➕ 출석 등록</div></div>
    <div class="form-row"><label class="lbl">날짜 *</label><input class="inp" type="date" id="att-date" style="max-width:200px"></div>
    <div class="form-row"><label class="lbl">출석 인원</label>
      <div id="att-p-box" style="padding:8px;background:var(--bg);border:1.5px solid var(--bd);border-radius:12px;max-height:150px;overflow-y:auto;display:flex;flex-direction:column;gap:2px"></div>
    </div>
    <div class="form-row"><label class="lbl">결석 인원</label>
      <div id="att-a-box" style="padding:8px;background:var(--bg);border:1.5px solid var(--bd);border-radius:12px;max-height:120px;overflow-y:auto;display:flex;flex-direction:column;gap:2px"></div>
    </div>
    <div class="form-row"><label class="lbl">메모</label><input class="inp" type="text" id="att-note" placeholder="메모 (선택)"></div>
    <button class="btn bp" onclick="addAttendance()">📊 출석 등록</button>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📊 출석 기록</div></div>
    <div class="list" id="att-admin-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-drive">
  <div class="p-t">공유 드라이브</div><div class="p-s">파일 링크를 등록하고 권한별로 관리합니다.</div>
  <div class="card"><div class="card-h"><div class="card-t">➕ 파일 등록</div></div>
    <div class="form-row"><label class="lbl">파일명 *</label><input class="inp" type="text" id="dr-name" placeholder="예: 촬영계획서_2026.pdf"></div>
    <div class="form-row"><label class="lbl">파일 URL *</label><input class="inp" type="url" id="dr-url" placeholder="https://..."></div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px">
      <div class="form-row"><label class="lbl">카테고리</label>
        <select class="inp sel" id="dr-cat"><option>양식</option><option>기획서</option><option>참고자료</option><option>기타</option></select>
      </div>
      <div class="form-row"><label class="lbl">최소 접근 역할 *</label>
        <select class="inp sel" id="dr-role">
          <option value="student">🎓 학생 이상</option><option value="member">👤 멤버 이상</option>
          <option value="staff">📸 부서원 이상</option><option value="teacher">👩‍🏫 담당선생님 이상</option>
          <option value="faculty">🏫 교직원 이상</option><option value="admin">🛡️ 관리자만</option>
        </select>
      </div>
    </div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px">
      <div class="form-row"><label class="lbl">파일 크기</label><input class="inp" type="text" id="dr-size" placeholder="예: 2.4MB"></div>
      <div class="form-row"><label class="lbl">설명</label><input class="inp" type="text" id="dr-desc" placeholder="파일 설명 (선택)"></div>
    </div>
    <button class="btn bp" onclick="addDrive()">📁 파일 등록</button>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📁 파일 목록</div></div>
    <div class="list" id="drive-admin-list"></div>
  </div>
</div>


<div class="tab-c" id="tab-online">
  <div class="p-t">온라인 세션</div><div class="p-s">현재 접속 중인 계정과 기기를 관리합니다.</div>
  <div class="card"><div class="card-h"><div class="card-t">🟢 현재 온라인</div><button class="btn bsm bg2" onclick="loadOnline()">새로고침</button></div>
    <div class="list" id="online-list"></div>
  </div>
</div>

<div class="tab-c" id="tab-settings">
  <div class="p-t">서버 설정</div><div class="p-s">점검 모드, 접근 차단 등을 관리합니다.</div>
  <div class="card"><div class="card-h"><div class="card-t">🔧 서버 점검 모드</div></div>
    <p style="font-size:13px;color:var(--mu);margin-bottom:14px;line-height:1.7">점검 모드 활성화 시 비관리자는 설정한 시간에 접근이 차단됩니다.</p>
    <div class="tog-row">
      <div><div class="tog-l">🔧 점검 모드 ON/OFF</div><div class="tog-s">비관리자 점검시간 차단</div></div>
      <label class="toggle"><input type="checkbox" id="cfg-maint" onchange="saveCfg()"><span class="tsl"></span></label>
    </div>
    <div id="maint-time" style="display:none;margin-top:12px">
      <div style="font-size:12px;font-weight:700;color:var(--mu);margin-bottom:8px">점검 시간 (미입력 시 항상 차단)</div>
      <div style="display:grid;grid-template-columns:1fr auto 1fr;gap:8px;align-items:end">
        <div><label class="lbl">시작</label><input class="inp" type="time" id="cfg-ms" onchange="saveCfg()"></div>
        <div style="padding-bottom:12px;color:var(--mu);font-weight:700">~</div>
        <div><label class="lbl">종료</label><input class="inp" type="time" id="cfg-me" onchange="saveCfg()"></div>
      </div>
    </div>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">🚫 즉시 접근 차단</div></div>
    <p style="font-size:13px;color:var(--mu);margin-bottom:14px;line-height:1.7">시간 설정 없이 즉각 비관리자 접근을 차단합니다.</p>
    <div class="tog-row">
      <div><div class="tog-l" style="color:#dc2626">🚫 즉시 접근 차단</div><div class="tog-s">비관리자 즉시 차단 (시간 무관)</div></div>
      <label class="toggle"><input type="checkbox" id="cfg-block" onchange="saveCfg()"><span class="tsl"></span></label>
    </div>
    <div id="block-warn" style="display:none;margin-top:10px;padding:10px 12px;border-radius:12px;background:#fff5f5;border:1.5px solid #fca5a5;font-size:13px;color:#dc2626;font-weight:600">
      ⚠️ 현재 접근 차단이 활성화되어 있습니다!
    </div>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📝 가입 신청 설정</div></div>
    <div class="tog-row">
      <div><div class="tog-l">가입 신청 받지 않기</div><div class="tog-s">공개 회원가입 신청 비활성화</div></div>
      <label class="toggle"><input type="checkbox" id="cfg-regclose" onchange="saveCfg()"><span class="tsl"></span></label>
    </div>
  </div>
  <div class="card"><div class="card-h"><div class="card-t">📊 현재 상태</div></div>
    <div id="srv-status" style="font-size:13px;color:var(--mu);line-height:2">불러오는 중...</div>
  </div>
</div>

</div>

<!-- 승인 모달 -->
<div class="modal" id="m-ap">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-ap')">✕</button>
    <div class="m-t">✅ 가입 신청 승인</div>
    <div class="m-s">아이디, 비밀번호, 역할을 설정하세요.</div>
    <div class="ipre" id="ap-info"></div>
    <input type="hidden" id="ap-id">
    <div class="form-row"><label class="lbl">아이디 *</label><input class="inp" type="text" id="ap-un" placeholder="사용자 아이디"></div>
    <div class="form-row"><label class="lbl">비밀번호 *</label><div class="pw-box"><input class="inp" type="password" id="ap-pw" placeholder="초기 비밀번호"><button type="button" class="pw-toggle" onclick="togglePw('ap-pw',this)">보기</button></div></div>
    <div class="form-row"><label class="lbl">역할 *</label>
      <select class="inp sel" id="ap-role">
        <option value="student">🎓 학생</option><option value="member">👤 멤버</option>
        <option value="staff">📸 부서원</option><option value="teacher">👩‍🏫 담당선생님</option><option value="bujang">👑 부장</option>
        <option value="faculty">🏫 교직원</option><option value="admin">🛡️ 관리자</option>
      </select>
    </div>
    <div style="background:#fffbeb;border:1.5px solid #fde68a;border-radius:12px;padding:13px;margin-bottom:14px">
      <div style="font-size:12px;font-weight:700;color:#92400e;margin-bottom:10px">🔒 관리자 본인 인증</div>
      <div class="form-row" style="margin-bottom:10px"><label class="lbl">관리자 비밀번호 *</label><div class="pw-box"><input class="inp" type="password" id="ap-adminpw" placeholder="현재 관리자 비밀번호"><button type="button" class="pw-toggle" onclick="togglePw('ap-adminpw',this)">보기</button></div></div>
      <div class="g-recaptcha" data-sitekey="${RECAPTCHA_SITE}" data-callback="onApCap" data-expired-callback="onApCapE"></div>
    </div>
    <div style="display:flex;gap:8px"><button class="btn bp" style="flex:1" onclick="doAp()" id="ap-btn">✅ 승인</button><button class="btn bg2" onclick="cModal('m-ap')">취소</button></div>
  </div>
</div>
<!-- 거절 모달 -->
<div class="modal" id="m-deny">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-deny')">✕</button>
    <div class="m-t">❌ 가입 신청 거절</div>
    <div class="m-s" id="deny-s"></div>
    <input type="hidden" id="deny-id">
    <div style="background:#fff5f5;border:1.5px solid #fca5a5;border-radius:12px;padding:13px;margin-bottom:14px">
      <div style="font-size:12px;font-weight:700;color:#991b1b;margin-bottom:10px">🔒 관리자 본인 인증</div>
      <div class="form-row" style="margin-bottom:0"><label class="lbl">관리자 비밀번호 *</label><div class="pw-box"><input class="inp" type="password" id="deny-adminpw" placeholder="현재 관리자 비밀번호"><button type="button" class="pw-toggle" onclick="togglePw('deny-adminpw',this)">보기</button></div></div>
    </div>
    <div style="display:flex;gap:8px"><button class="btn bdanger" style="flex:1" onclick="doDeny()" id="deny-btn">❌ 거절 확인</button><button class="btn bg2" onclick="cModal('m-deny')">취소</button></div>
  </div>
</div>
<!-- 멤버 수정 모달 -->
<div class="modal" id="m-edit">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-edit')">✕</button>
    <div class="m-t">✏️ 멤버 정보 수정</div>
    <input type="hidden" id="edit-id">
    <div class="form-row"><label class="lbl">이름</label><input class="inp" type="text" id="edit-name"></div>
    <div class="form-row"><label class="lbl">아이디</label><input class="inp" type="text" id="edit-un"></div>
    <div class="form-row"><label class="lbl">역할</label>
      <select class="inp sel" id="edit-role">
        <option value="student">🎓 학생</option><option value="member">👤 멤버</option>
        <option value="staff">📸 부서원</option><option value="teacher">👩‍🏫 담당선생님</option><option value="bujang">👑 부장</option>
        <option value="faculty">🏫 교직원</option><option value="admin">🛡️ 관리자</option>
      </select>
    </div>
    <div class="tog-row">
      <div><div class="tog-l">관리자 권한 ON/OFF</div><div class="tog-s">역할과 별개로 관리자 패널 접근 권한</div></div>
      <label class="toggle"><input type="checkbox" id="edit-admin"><span class="tsl"></span></label>
    </div>
    <div style="display:flex;gap:8px;margin-top:14px"><button class="btn bp" style="flex:1" onclick="doEdit()">저장</button><button class="btn bg2" onclick="cModal('m-edit')">취소</button></div>
  </div>
</div>
<!-- 비번 재설정 모달 -->
<div class="modal" id="m-rp">
  <div class="mbox">
    <button class="m-x" onclick="cModal('m-rp')">✕</button>
    <div class="m-t">🔑 비밀번호 재설정</div>
    <div class="m-s" id="rp-s"></div>
    <input type="hidden" id="rp-id">
    <div class="form-row"><label class="lbl">새 비밀번호 *</label><div class="pw-box"><input class="inp" type="password" id="rp-pw" placeholder="새 비밀번호"><button type="button" class="pw-toggle" onclick="togglePw('rp-pw',this)">보기</button></div></div>
    <div style="display:flex;gap:8px"><button class="btn bp" style="flex:1" onclick="doRp()">🔑 변경</button><button class="btn bg2" onclick="cModal('m-rp')">취소</button></div>
  </div>
</div>

<div class="toast-wrap" id="tw"></div>
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
${commonJS()}
<script>
var pend=[],mems=[],scheds=[],notices=[],attRecs=[],drives=[];
var apCap=null;
window.onApCap=function(t){apCap=t;}
window.onApCapE=function(){apCap=null;}
var RLBL={student:'학생',member:'멤버',staff:'부서원',teacher:'담당선생님',bujang:'부장',faculty:'교직원',admin:'관리자'};
var ROLE_ORDER=['student','member','staff','teacher','bujang','faculty','admin'];
function rlvl(r){return ROLE_ORDER.indexOf(r);}

function showTab(id,btn){
  document.querySelectorAll('.tab-c').forEach(function(x){x.classList.remove('active');});
  document.querySelectorAll('.sbi').forEach(function(x){x.classList.remove('active');});
  document.getElementById('tab-'+id).classList.add('active');
  if(btn)btn.classList.add('active');
  if(id==='pend')loadPend();
  if(id==='mems')loadMems();
  if(id==='scheds'){loadMems();loadScheds();}
  if(id==='notices')loadNotices();
  if(id==='att'){loadMems();loadAtt();}
  if(id==='drive')loadDrives();
  if(id==='online')loadOnline();
  if(id==='settings')loadCfg();
}


async function bulkDeny(){
  var ids=[]; document.querySelectorAll('.pend-chk:checked').forEach(function(c){ ids.push(c.value); });
  if(!ids.length){ toast('선택된 신청이 없습니다.','e'); return; }
  var r=await fetch('/api/bulk-deny',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({ids:ids})}); var d=await r.json();
  if(d.success){ toast('일괄 거절 완료','s'); loadPend(); } else toast('실패','e');
}
async function toggleBlock(userId,blocked){
  var r=await fetch('/api/toggle-block-user',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({userId:userId,blocked:blocked})}); var d=await r.json();
  if(d.success){ toast(blocked?'차단 완료':'차단 해제 완료',blocked?'e':'s'); loadMems(); } else toast(d.message||'실패','e');
}
async function loadOnline(){
  try{
    var r=await fetch('/api/online-sessions'); var d=await r.json();
    var el=document.getElementById('online-list'); el.innerHTML='';
    (d.sessions||[]).forEach(function(s){
      var li=document.createElement('div'); li.className='li';
      li.innerHTML='<div class="li-b"><div class="li-t">'+e(s.name)+' ('+e(s.username)+')</div><div class="li-m">'+e(s.ip)+' · '+e(s.device)+' · '+e(s.role)+' · 로그인 '+e(s.loginAt)+'</div></div>';
      var acts=document.createElement('div'); acts.className='li-a';
      var a=document.createElement('button'); a.className='btn bdanger bsm'; a.textContent='세션 종료'; a.onclick=function(){ forceLogout({sid:s.sid}); };
      var b=document.createElement('button'); b.className='btn bg2 bsm'; b.textContent='사용자 전체 종료'; b.onclick=function(){ forceLogout({userId:s.userId}); };
      acts.appendChild(a); acts.appendChild(b); li.appendChild(acts); el.appendChild(li);
    });
    if(!(d.sessions||[]).length) el.innerHTML='<div class="empty"><div class="empty-ic">🟢</div>현재 온라인 세션이 없습니다.</div>';
  }catch(ex){}
}
async function forceLogout(payload){
  var r=await fetch('/api/force-logout',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(payload)}); var d=await r.json();
  if(d.success){ toast('세션 종료 완료','s'); loadOnline(); } else toast('실패','e');
}

/* ── 신청 관리 ── */
async function loadPend(){
  try{
    var r=await fetch('/api/pending');var d=await r.json();pend=d.pending||[];
    renderPend();
    document.getElementById('d-pend').textContent=String(pend.length);
    document.getElementById('pb').textContent=String(pend.length);
  }catch(ex){toast('신청 목록 불러오기 실패','e');}
}
function renderPend(){
  var list=document.getElementById('pend-list');
  var dash=document.getElementById('d-pend-list');
  document.getElementById('pend-cnt').textContent=String(pend.length)+'건';
  list.innerHTML='';dash.innerHTML='';
  if(!pend.length){
    var em=document.createElement('div');em.className='empty';em.innerHTML='<div class="empty-ic">✅</div>대기 중인 신청이 없습니다.';
    list.appendChild(em.cloneNode(true));dash.appendChild(em);return;
  }
  function makeItem(p){
    var wrap=document.createElement('div');wrap.className='li'; var ck=document.createElement('input'); ck.type='checkbox'; ck.className='pend-chk'; ck.value=p.id; ck.style.marginRight='8px';
    var badge=document.createElement('span');
    badge.style.cssText='flex-shrink:0;font-size:10px;font-weight:700;padding:3px 8px;border-radius:100px;background:#fef3c7;color:#92400e;border:1px solid #fde68a;white-space:nowrap';
    badge.textContent='대기';
    var body=document.createElement('div');body.className='li-b';
    var tl=document.createElement('div');tl.className='li-t';
    tl.textContent=p.name+' ';
    var sub=document.createElement('span');sub.style.cssText='font-size:11px;color:var(--mu)';
    sub.textContent=(p.regType==='teacher'?'선생님':'학생')+(p.grade?' · '+p.grade+'학년 '+p.classNum+'반':'');
    tl.appendChild(sub);
    var meta=document.createElement('div');meta.className='li-m';
    meta.textContent=(p.regType==='teacher'&&p.desiredUsername?'ID: '+p.desiredUsername:'TEL '+p.phone)+(p.subject?' · '+p.subject:'')+' · '+p.requestedAt;
    body.appendChild(tl);body.appendChild(meta);
    var acts=document.createElement('div');acts.className='li-a';
    var b1=document.createElement('button');b1.className='btn bp bsm';b1.textContent='승인';
    b1.onclick=function(){oAp(p.id);};
    var b2=document.createElement('button');b2.className='btn bdanger bsm';b2.textContent='거절';
    b2.onclick=function(){oDeny(p.id,p.name);};
    acts.appendChild(b1);acts.appendChild(b2);
    wrap.appendChild(ck);wrap.appendChild(badge);wrap.appendChild(body);wrap.appendChild(acts);
    return wrap;
  }
  pend.forEach(function(p){list.appendChild(makeItem(p));});
  pend.slice(0,3).forEach(function(p){dash.appendChild(makeItem(p));});
}
function oAp(id){
  var p=pend.find(function(x){return x.id===id;});if(!p)return;
  document.getElementById('ap-id').value=id;
  document.getElementById('ap-un').value=(p.desiredUsername||'');document.getElementById('ap-pw').value='';document.getElementById('ap-adminpw').value='';
  apCap=null;if(window.grecaptcha)grecaptcha.reset();
  var info=document.getElementById('ap-info');info.innerHTML='';
  function row(k,v){info.innerHTML+='<div class="ir"><div class="ik">'+k+'</div><div class="iv">'+e(v)+'</div></div>';}
  row('이름',p.name);row('유형',p.regType==='teacher'?'선생님':'학생');
  if(p.grade)row('학년/반',p.grade+'학년 '+p.classNum+'반 '+p.attendanceNum+'번');
  if(p.phone)row('전화번호',p.phone);if(p.desiredUsername)row('희망 ID',p.desiredUsername);
  if(p.subject)row('담당과목',p.subject);if(p.position)row('직책',p.position);
  if(p.desiredUsername){
    var note=document.createElement('div');
    note.style.cssText='margin-top:8px;padding:8px 10px;background:#f0fdf4;border:1px solid #86efac;border-radius:8px;font-size:11px;color:#166534';
    note.textContent='\ud83d\udcdd \uc774 \uad50\uc0ac\ub294 \uc544\uc774\ub514/\ube44\ubc00\ubc88\ud638\ub97c \uc9c1\uc811 \uc124\uc815\ud588\uc2b5\ub2c8\ub2e4. \ube44\ubc00\ubc88\ud638 \uce78\uc744 \ube44\uc6cc\ub450\uba74 \uad50\uc0ac\uac00 \uc124\uc815\ud55c \ube44\ubc00\ubc88\ud638\uac00 \uc0ac\uc6a9\ub429\ub2c8\ub2e4.';
    info.appendChild(note);
  }
  oModal('m-ap');
  /* 5-second countdown for approve */
  var apBtn=document.getElementById('ap-btn');
  if(apBtn){
    apBtn.disabled=true;
    var apRemain=5;
    apBtn.textContent='\u2705 \uC2B9\uC778 ('+apRemain+')';
    var apTimer=setInterval(function(){
      apRemain--;
      if(apRemain>0){apBtn.textContent='\u2705 \uC2B9\uC778 ('+apRemain+')';}
      else{apBtn.textContent='\u2705 \uC2B9\uC778';apBtn.disabled=false;clearInterval(apTimer);}
    },1000);
  }
}
async function doAp(){
  var pid=document.getElementById('ap-id').value,un=document.getElementById('ap-un').value.trim();
  var pw=document.getElementById('ap-pw').value.trim(),role=document.getElementById('ap-role').value;
  var adminPw=document.getElementById('ap-adminpw').value;
  if(!un){toast('아이디를 입력하세요.','e');return;}
  if(!adminPw){toast('관리자 비밀번호를 입력하세요.','e');return;}
  if(!apCap){toast('reCAPTCHA 인증을 완료해주세요.','e');return;}
  try{
    var r=await fetch('/api/approve',{method:'POST',headers:{'Content-Type':'application/json'},
      body:JSON.stringify({pendingId:pid,username:un,password:pw,role:role,adminPw:adminPw,captchaToken:apCap})});
    var d=await r.json();
    if(d.success){toast('승인 완료!','s');cModal('m-ap');loadPend();loadMems();}
    else toast(d.message||'승인 실패','e');
  }catch(ex){toast('서버 오류','e');}
}
function oDeny(id,name){
  document.getElementById('deny-id').value=id;
  document.getElementById('deny-s').textContent=name+'\uB2D8\uC758 \uC2E0\uCCAD\uC744 \uAC70\uC808\uD558\uC2DC\uACA0\uC2B5\uB2C8\uAE4C?';
  document.getElementById('deny-adminpw').value='';
  /* 5-second countdown for deny */
  var denyBtn=document.getElementById('deny-btn');
  if(denyBtn){
    denyBtn.disabled=true;
    var denyRemain=5;
    denyBtn.textContent='\u274C \uAC70\uC808 ('+denyRemain+')';
    var denyTimer=setInterval(function(){
      denyRemain--;
      if(denyRemain>0){denyBtn.textContent='\u274C \uAC70\uC808 ('+denyRemain+')';}
      else{denyBtn.textContent='\u274C \uAC70\uC808';denyBtn.disabled=false;clearInterval(denyTimer);}
    },1000);
  }
  oModal('m-deny');
}
async function doDeny(){
  var id=document.getElementById('deny-id').value,adminPw=document.getElementById('deny-adminpw').value;
  if(!adminPw){toast('관리자 비밀번호를 입력하세요.','e');return;}
  try{
    var r=await fetch('/api/pending/'+id,{method:'DELETE'});var d=await r.json();
    if(d.success){toast('거절 처리됐습니다.','s');cModal('m-deny');loadPend();}
    else toast('처리 실패','e');
  }catch(ex){toast('서버 오류','e');}
}

/* ── 멤버 관리 ── */
async function loadMems(){
  try{
    var r=await fetch('/api/members');var d=await r.json();mems=d.members||[];
    renderMems(mems);document.getElementById('d-mems').textContent=String(mems.length);
    renderAssigneeBox();renderAttBoxes();
  }catch(ex){}
}
function filterMems(q){
  q=q.toLowerCase();
  renderMems(mems.filter(function(m){return m.name.toLowerCase().indexOf(q)>=0||m.username.toLowerCase().indexOf(q)>=0;}));
}
function renderMems(list){
  document.getElementById('mems-cnt').textContent=String(list.length)+'명';
  var el=document.getElementById('mems-list');el.innerHTML='';
  if(!list.length){el.innerHTML='<div class="empty"><div class="empty-ic">-</div>멤버가 없습니다.</div>';return;}
  list.forEach(function(m){
    var bcMap={admin:'background:#e0e7ff;color:#4338ca',staff:'background:#d1fae5;color:#047857',student:'background:#dbeafe;color:#1d4ed8',teacher:'background:#ede9fe;color:#7c3aed',faculty:'background:#fef3c7;color:#92400e',member:'background:#d1fae5;color:#065f46'};
    var bc=bcMap[m.role]||'background:#d8f3dc;color:#2d6a4f';
    var li=document.createElement('div');li.className='li';
    var badge=document.createElement('span');
    badge.style.cssText='flex-shrink:0;font-size:10px;font-weight:700;padding:3px 8px;border-radius:100px;white-space:nowrap;'+bc;
    badge.textContent=(RLBL[m.role]||m.role)+(m.isAdmin&&m.role!=='admin'?' *':'');
    var body=document.createElement('div');body.className='li-b';
    var t=document.createElement('div');t.className='li-t';t.textContent=m.name;
    var mt=document.createElement('div');mt.className='li-m';mt.textContent='ID: '+m.username+(m.phone?' · TEL '+m.phone:'')+(m.createdAt?' · '+m.createdAt:'');
    body.appendChild(t);body.appendChild(mt);
    var acts=document.createElement('div');acts.className='li-a';
    var b1=document.createElement('button');b1.className='btn bg2 bsm';b1.textContent='수정';b1.onclick=function(){oEdit(m.id);};
    var b2=document.createElement('button');b2.className='btn bg2 bsm';b2.textContent='비번';b2.onclick=function(){oRp(m.id,m.name);};
    var b4=document.createElement('button');b4.className='btn bsm '+(m.blocked?'bg2':'bdanger');b4.textContent=m.blocked?'차단해제':'차단';b4.onclick=function(){toggleBlock(m.id,!m.blocked);};
    acts.appendChild(b1);acts.appendChild(b2);acts.appendChild(b4);
    if(m.id!=='admin'){
      var b3=document.createElement('button');b3.className='btn bdanger bsm';b3.textContent='삭제';
      b3.onclick=function(){delMem(m.id,m.name);};acts.appendChild(b3);
    }
    li.appendChild(badge);li.appendChild(body);li.appendChild(acts);el.appendChild(li);
  });
}
function oEdit(id){
  var m=mems.find(function(x){return x.id===id;});if(!m)return;
  document.getElementById('edit-id').value=id;document.getElementById('edit-name').value=m.name;
  document.getElementById('edit-un').value=m.username;document.getElementById('edit-role').value=m.role||'member';
  document.getElementById('edit-admin').checked=!!m.isAdmin;oModal('m-edit');
}
async function doEdit(){
  var uid=document.getElementById('edit-id').value,name=document.getElementById('edit-name').value.trim();
  var un=document.getElementById('edit-un').value.trim(),role=document.getElementById('edit-role').value;
  var ia=document.getElementById('edit-admin').checked;
  try{
    var r1=await fetch('/api/update-user',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({userId:uid,username:un,role:role,name:name})});
    var d1=await r1.json();if(!d1.success){toast(d1.message||'수정 실패','e');return;}
    var r2=await fetch('/api/toggle-admin',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({userId:uid,isAdmin:ia})});
    var d2=await r2.json();if(!d2.success){toast(d2.message||'권한 변경 실패','e');return;}
    toast('✅ 수정 완료','s');cModal('m-edit');loadMems();
  }catch(ex){toast('서버 오류','e');}
}
function delMem(id,name){
  sjyConfirm(name+'님 계정을 삭제합니다',function(){
    fetch('/api/member/'+id,{method:'DELETE'}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast(name+'님 삭제 완료','s');loadMems();}else toast(d.message||'삭제 실패','e');
    }).catch(function(){toast('서버 오류','e');});
  },{danger:true,sub:'이 작업은 되돌릴 수 없습니다.',okText:'삭제',cancelText:'취소'});
}
function oRp(id,name){
  document.getElementById('rp-id').value=id;document.getElementById('rp-pw').value='';
  document.getElementById('rp-s').textContent=name+'님의 비밀번호를 재설정합니다.';oModal('m-rp');
}
async function doRp(){
  var uid=document.getElementById('rp-id').value,pw=document.getElementById('rp-pw').value.trim();
  if(!pw){toast('새 비밀번호를 입력하세요.','e');return;}
  try{
    var r=await fetch('/api/reset-pw',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({userId:uid,newPassword:pw})});
    var d=await r.json();
    if(d.success){toast('비번 변경 완료','s');cModal('m-rp');}else toast(d.message||'실패','e');
  }catch(ex){toast('서버 오류','e');}
}

/* ── 담당자/출석 체크박스 ── */
function renderAssigneeBox(){
  var box=document.getElementById('assignee-box');box.innerHTML='';
  if(!mems.length){box.innerHTML='<div style="font-size:12px;color:var(--mu);padding:6px">멤버를 먼저 불러오세요.</div>';return;}
  mems.forEach(function(m){
    var row=document.createElement('div');row.className='mem-chk-row';
    var lbl=document.createElement('label');lbl.style.cssText='display:flex;align-items:center;gap:8px;cursor:pointer;width:100%';
    var chk=document.createElement('input');chk.type='checkbox';chk.className='asgn-chk';chk.value=m.name;chk.dataset.id=m.id;
    var nm=document.createElement('span');nm.style.cssText='font-size:13px;color:var(--p)';nm.textContent=m.name;
    var rl=document.createElement('span');rl.style.cssText='font-size:11px;color:var(--mu)';rl.textContent=RLBL[m.role]||m.role;
    lbl.appendChild(chk);lbl.appendChild(nm);lbl.appendChild(rl);row.appendChild(lbl);box.appendChild(row);
  });
}
function renderAttBoxes(){
  function fill(boxId){
    var box=document.getElementById(boxId);box.innerHTML='';
    if(!mems.length){box.innerHTML='<div style="font-size:12px;color:var(--mu);padding:6px">멤버 없음</div>';return;}
    mems.forEach(function(m){
      var row=document.createElement('div');row.className='mem-chk-row';
      var lbl=document.createElement('label');lbl.style.cssText='display:flex;align-items:center;gap:8px;cursor:pointer;width:100%';
      var chk=document.createElement('input');chk.type='checkbox';chk.value=m.name;
      var nm=document.createElement('span');nm.style.cssText='font-size:13px;color:var(--p)';nm.textContent=m.name;
      var rl=document.createElement('span');rl.style.cssText='font-size:11px;color:var(--mu)';rl.textContent=RLBL[m.role]||m.role;
      lbl.appendChild(chk);lbl.appendChild(nm);lbl.appendChild(rl);row.appendChild(lbl);box.appendChild(row);
    });
  }
  fill('att-p-box');fill('att-a-box');
}

/* ── 일정 ── */
async function loadScheds(){
  try{
    var r=await fetch('/api/schedules');var d=await r.json();scheds=d.schedules||[];
    document.getElementById('d-schs').textContent=String(scheds.length);
    var el=document.getElementById('sch-admin-list');el.innerHTML='';
    if(!scheds.length){el.innerHTML='<div class="empty"><div class="empty-ic">📅</div>등록된 일정이 없습니다.</div>';return;}
    var TL={shoot:'📷 촬영',club:'🎯 동아리',meeting:'💼 회의'};
    var BC={shoot:'background:#fef3c7;color:#92400e',club:'background:#ede9fe;color:#7c3aed',meeting:'background:#dbeafe;color:#1d4ed8'};
    scheds.forEach(function(s){
      var li=document.createElement('div');li.className='li';
      var badge=document.createElement('span');
      badge.style.cssText='flex-shrink:0;font-size:10px;font-weight:700;padding:3px 8px;border-radius:100px;white-space:nowrap;'+(BC[s.type]||'');
      badge.textContent=TL[s.type]||s.type;
      var body=document.createElement('div');body.className='li-b';
      var t=document.createElement('div');t.className='li-t';t.textContent=s.title+' '+s.date+(s.time?' '+s.time:'');
      var mt=document.createElement('div');mt.className='li-m';mt.textContent=(s.note?s.note+' · ':'')+s.createdBy;
      body.appendChild(t);body.appendChild(mt);
      if(s.assignees&&s.assignees.length){var a=document.createElement('div');a.className='li-m';a.textContent='담당: '+s.assignees.join(', ');body.appendChild(a);}
      var acts=document.createElement('div');acts.className='li-a';
      var btn=document.createElement('button');btn.className='btn bdanger bsm';btn.textContent='삭제';
      btn.onclick=function(){delSch(s.id);};acts.appendChild(btn);
      li.appendChild(badge);li.appendChild(body);li.appendChild(acts);el.appendChild(li);
    });
  }catch(ex){}
}
async function addSchedule(){
  var type=document.getElementById('sch-type').value,title=document.getElementById('sch-title').value.trim();
  var date=document.getElementById('sch-date').value,time=document.getElementById('sch-time').value;
  var note=document.getElementById('sch-note').value.trim();
  if(!title||!date){toast('제목과 날짜를 입력하세요.','e');return;}
  var asgn=[],aIds=[];
  document.querySelectorAll('.asgn-chk:checked').forEach(function(c){asgn.push(c.value);aIds.push(c.dataset.id);});
  try{
    var r=await fetch('/api/schedule',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({title:title,type:type,date:date,time:time,note:note,assignees:asgn,assigneeIds:aIds})});
    var d=await r.json();
    if(d.success){
      toast('일정 등록 완료','s');
      document.getElementById('sch-title').value='';document.getElementById('sch-date').value='';
      document.getElementById('sch-time').value='';document.getElementById('sch-note').value='';
      document.querySelectorAll('.asgn-chk').forEach(function(c){c.checked=false;});
      loadScheds();
    }else toast(d.message||'실패','e');
  }catch(ex){toast('서버 오류','e');}
}
function delSch(id){
  sjyConfirm('이 일정을 삭제할까요?',function(){
    fetch('/api/schedule/'+id,{method:'DELETE'}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast('삭제 완료','s');loadScheds();}else toast('삭제 실패','e');
    }).catch(function(){toast('서버 오류','e');});
  },{danger:true,okText:'삭제',cancelText:'취소'});
}

/* ── 공지 ── */
async function loadNotices(){
  try{
    var r=await fetch('/api/notices');var d=await r.json();notices=d.notices||[];
    document.getElementById('d-noti').textContent=String(notices.length);
    var el=document.getElementById('notice-admin-list');el.innerHTML='';
    if(!notices.length){el.innerHTML='<div class="empty"><div class="empty-ic">📢</div>게시물이 없습니다.</div>';return;}
    notices.forEach(function(n){
      var li=document.createElement('div');li.className='li';
      var badge=document.createElement('span');
      badge.style.cssText='flex-shrink:0;font-size:10px;font-weight:700;padding:3px 8px;border-radius:100px;white-space:nowrap;'+(n.noticeType==='update'?'background:#ede9fe;color:#7c3aed':'background:#dbeafe;color:#1d4ed8');
      badge.textContent=n.noticeType==='update'?'🆕 업데이트':'📢 공지';
      var body=document.createElement('div');body.className='li-b';
      var t=document.createElement('div');t.className='li-t';t.textContent=n.title;
      var mt=document.createElement('div');mt.className='li-m';mt.textContent=n.author+' · '+n.createdAt;
      body.appendChild(t);body.appendChild(mt);
      var acts=document.createElement('div');acts.className='li-a';
      var btn=document.createElement('button');btn.className='btn bdanger bsm';btn.textContent='삭제';
      btn.onclick=function(){delNotice(n.id);};acts.appendChild(btn);
      li.appendChild(badge);li.appendChild(body);li.appendChild(acts);el.appendChild(li);
    });
  }catch(ex){}
}
async function addNotice(){
  var type=document.getElementById('noti-type').value,title=document.getElementById('noti-title').value.trim();
  var content=document.getElementById('noti-content').value.trim();
  if(!title||!content){toast('제목과 내용을 입력하세요.','e');return;}
  try{
    var r=await fetch('/api/notice',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({title:title,content:content,noticeType:type})});
    var d=await r.json();
    if(d.success){toast('게시 완료! 전체 알림 발송됨','s');document.getElementById('noti-title').value='';document.getElementById('noti-content').value='';loadNotices();}
    else toast(d.message||'실패','e');
  }catch(ex){toast('서버 오류','e');}
}
function delNotice(id){
  sjyConfirm('이 게시물을 삭제할까요?',function(){
    fetch('/api/notice/'+id,{method:'DELETE'}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast('삭제 완료','s');loadNotices();}else toast('실패','e');
    }).catch(function(){toast('서버 오류','e');});
  },{danger:true,okText:'삭제',cancelText:'취소'});
}

/* ── 출석 ── */
async function loadAtt(){
  try{
    var r=await fetch('/api/attendance');var d=await r.json();attRecs=d.records||[];
    var el=document.getElementById('att-admin-list');el.innerHTML='';
    if(!attRecs.length){el.innerHTML='<div class="empty"><div class="empty-ic">📊</div>출석 기록이 없습니다.</div>';return;}
    attRecs.forEach(function(rec){
      var li=document.createElement('div');li.className='li';
      var body=document.createElement('div');body.className='li-b';
      var t=document.createElement('div');t.className='li-t';t.textContent=rec.date;
      var mt=document.createElement('div');mt.className='li-m';mt.textContent='출석: '+((rec.attendees||[]).join(', ')||'없음')+' / 결석: '+((rec.absentees||[]).join(', ')||'없음');
      body.appendChild(t);body.appendChild(mt);
      if(rec.note){var nt=document.createElement('div');nt.className='li-m';nt.textContent=rec.note;body.appendChild(nt);}
      var acts=document.createElement('div');acts.className='li-a';
      var btn=document.createElement('button');btn.className='btn bdanger bsm';btn.textContent='삭제';
      btn.onclick=function(){delAtt(rec.id);};acts.appendChild(btn);
      li.appendChild(body);li.appendChild(acts);el.appendChild(li);
    });
  }catch(ex){}
}
async function addAttendance(){
  var date=document.getElementById('att-date').value,note=document.getElementById('att-note').value.trim();
  if(!date){toast('날짜를 입력하세요.','e');return;}
  var att=[],abs=[];
  document.querySelectorAll('#att-p-box input:checked').forEach(function(c){att.push(c.value);});
  document.querySelectorAll('#att-a-box input:checked').forEach(function(c){abs.push(c.value);});
  try{
    var r=await fetch('/api/attendance',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({date:date,attendees:att,absentees:abs,note:note})});
    var d=await r.json();
    if(d.success){toast('출석 등록 완료','s');document.getElementById('att-date').value='';document.getElementById('att-note').value='';document.querySelectorAll('#att-p-box input,#att-a-box input').forEach(function(c){c.checked=false;});loadAtt();}
    else toast(d.message||'실패','e');
  }catch(ex){toast('서버 오류','e');}
}
function delAtt(id){
  sjyConfirm('이 출석 기록을 삭제할까요?',function(){
    fetch('/api/attendance/'+id,{method:'DELETE'}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast('삭제 완료','s');loadAtt();}else toast('실패','e');
    }).catch(function(){toast('서버 오류','e');});
  },{danger:true,okText:'삭제',cancelText:'취소'});
}

/* ── 드라이브 ── */
async function loadDrives(){
  try{
    var r=await fetch('/api/drives');var d=await r.json();drives=d.drives||[];
    var el=document.getElementById('drive-admin-list');el.innerHTML='';
    if(!drives.length){el.innerHTML='<div class="empty"><div class="empty-ic">📁</div>등록된 파일이 없습니다.</div>';return;}
    drives.forEach(function(f){
      var li=document.createElement('div');li.className='li';
      var body=document.createElement('div');body.className='li-b';
      var t=document.createElement('div');t.className='li-t';t.textContent=f.filename+' ['+f.category+']';
      var mt=document.createElement('div');mt.className='li-m';mt.textContent='최소 권한: '+(RLBL[f.minRole]||f.minRole)+' · '+f.uploadedBy+' · '+f.uploadedAt;
      body.appendChild(t);body.appendChild(mt);
      var acts=document.createElement('div');acts.className='li-a';
      var a=document.createElement('a');a.href=f.url;a.target='_blank';a.className='btn bg2 bsm';a.textContent='링크';
      var btn=document.createElement('button');btn.className='btn bdanger bsm';btn.textContent='삭제';
      btn.onclick=function(){delDr(f.id);};
      acts.appendChild(a);acts.appendChild(btn);li.appendChild(body);li.appendChild(acts);el.appendChild(li);
    });
  }catch(ex){}
}
async function addDrive(){
  var nm=document.getElementById('dr-name').value.trim(),url=document.getElementById('dr-url').value.trim();
  var cat=document.getElementById('dr-cat').value,role=document.getElementById('dr-role').value;
  var desc=document.getElementById('dr-desc').value.trim(),size=document.getElementById('dr-size').value.trim();
  if(!nm||!url){toast('파일명과 URL을 입력하세요.','e');return;}
  try{
    var r=await fetch('/api/drive',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({filename:nm,url:url,category:cat,minRole:role,description:desc,filesize:size})});
    var d=await r.json();
    if(d.success){toast('파일 등록 완료','s');document.getElementById('dr-name').value='';document.getElementById('dr-url').value='';document.getElementById('dr-desc').value='';document.getElementById('dr-size').value='';loadDrives();}
    else toast(d.message||'실패','e');
  }catch(ex){toast('서버 오류','e');}
}
function delDr(id){
  sjyConfirm('이 파일을 목록에서 삭제할까요?',function(){
    fetch('/api/drive/'+id,{method:'DELETE'}).then(function(r){return r.json();}).then(function(d){
      if(d.success){toast('삭제 완료','s');loadDrives();}else toast('실패','e');
    }).catch(function(){toast('서버 오류','e');});
  },{danger:true,okText:'삭제',cancelText:'취소'});
}

/* ── 설정 ── */
async function loadCfg(){
  try{
    var r=await fetch('/api/config');var d=await r.json();var cfg=d.config||{};
    document.getElementById('cfg-maint').checked=!!cfg.maintenance;
    document.getElementById('cfg-block').checked=!!cfg.blockAccess;
    document.getElementById('cfg-ms').value=cfg.maintStart||'';
    document.getElementById('cfg-me').value=cfg.maintEnd||'';
    document.getElementById('cfg-regclose').checked=!!cfg.registerClosed;
    document.getElementById('maint-time').style.display=cfg.maintenance?'block':'none';
    document.getElementById('block-warn').style.display=cfg.blockAccess?'block':'none';
    document.getElementById('srv-status').innerHTML=
      '🔧 점검 모드: <strong style="color:'+(cfg.maintenance?'#166534':'#dc2626')+'">'+(cfg.maintenance?'활성화':'비활성화')+'</strong>'+(cfg.maintenance&&cfg.maintStart?' ('+cfg.maintStart+'~'+cfg.maintEnd+')':'')+'<br>'+
      '🚫 즉시 차단: <strong style="color:'+(cfg.blockAccess?'#dc2626':'#166534')+'">'+(cfg.blockAccess?'활성화 ⚠️':'비활성화')+'</strong><br>'+
      '📝 가입 신청: <strong style="color:'+(cfg.registerClosed?'#dc2626':'#166534')+'">'+(cfg.registerClosed?'받지 않음':'받는 중')+'</strong>';
  }catch(ex){toast('설정 불러오기 실패','e');}
}
async function saveCfg(){
  var maint=document.getElementById('cfg-maint').checked,block=document.getElementById('cfg-block').checked;
  var ms=document.getElementById('cfg-ms').value,me=document.getElementById('cfg-me').value,regClosed=document.getElementById('cfg-regclose').checked;
  document.getElementById('maint-time').style.display=maint?'block':'none';
  document.getElementById('block-warn').style.display=block?'block':'none';
  try{
    var r=await fetch('/api/config',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({maintenance:maint,blockAccess:block,maintStart:ms,maintEnd:me,registerClosed:regClosed})});
    var d=await r.json();
    if(d.success){toast(block?'⚠️ 접근 차단 활성화됨':'✅ 설정 저장됨',block?'e':'s');loadCfg();}
    else toast('저장 실패','e');
  }catch(ex){toast('서버 오류','e');}
}

loadPend();loadMems();loadScheds();loadNotices();
document.addEventListener('contextmenu',function(ev){ev.preventDefault();});
document.addEventListener('selectstart',function(ev){ev.preventDefault();});
</script></body></html>`;
}
