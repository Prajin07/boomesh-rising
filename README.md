
function updateWin(dt) {
  winT += dt; const p = Math.min(1, winT / 6);
  const white = new THREE.Color(0xfff2cc);
  scene.background.copy(new THREE.Color(W.d.bg)).lerp(white, p); scene.fog.color.copy(scene.background); scene.fog.density = lerp(W.d.fog, 0.004, p);
  ambient.color.set(0xffffff); ambient.intensity = lerp(W.d.ambI, 2.2, p); torch.intensity = lerp(1.5, 0.3, p);
  if (W.winLight) W.winLight.intensity = lerp(0, 5, p);
  W.boss.mesh.scale.setScalar(1.15 * (1 - p * 0.95)); W.boss.mesh.position.y = p * 6; if (p > 0.85) W.boss.mesh.visible = false;
  if (p >= 1 && $('endBox').classList.contains('hidden')) {
    $('endStats').textContent = 'Times you fell: ' + deaths + ' (' + HEROES[heroKey].name + ' led the way)';
    $('endBox').classList.remove('hidden');
  }
}

function updateHud() {
  $('hpFill').style.width = (P.hp / P.maxHp * 100) + '%';
  $('stFill').style.width = (P.stamina / CFG.staminaMax * 100) + '%';
  $('objective').textContent = 'Goal: ' + objectiveText();
  $('hurt').style.opacity = hurtFlash;
  // signal / compass
  let target = null, label = '';
  if (W.boss) { label = ''; }
  else if (W.collected >= CFG.hintsNeeded) { target = ALTAR; label = 'Altar: ' + Math.round(Math.hypot(ALTAR.x - P.pos.x, ALTAR.z - P.pos.z)) + ' m'; }
  else {
    let best = 1e9, bh = null; for (const h of W.hints) if (!h.got) { const dd = Math.hypot(h.x - P.pos.x, h.z - P.pos.z); if (dd < best) { best = dd; bh = h; } }
    if (bh) { const eff = best / hero.sense; label = 'Hint signal: ' + (eff < 12 ? 'BURNING' : eff < 26 ? 'warm' : eff < 45 ? 'cool' : 'cold'); if (hero.sense > 1) target = bh; }
  }
  $('signal').textContent = label;
  if (target) {
    const dx = target.x - P.pos.x, dz = target.z - P.pos.z, fx = -Math.sin(camYaw), fz = -Math.cos(camYaw), rx = Math.cos(camYaw), rz = -Math.sin(camYaw);
    $('arrow').style.transform = 'rotate(' + (Math.atan2(dx * rx + dz * rz, dx * fx + dz * fz) * 180 / Math.PI) + 'deg)'; $('compass').style.visibility = 'visible';
  } else $('compass').style.visibility = 'hidden';
}

let last = performance.now();
function loop(now) {
  requestAnimationFrame(loop);
  const dt = Math.min(0.05, (now - last) / 1000); last = now;
  if (W && hero) {
    if (state === 'play') updatePlayer(dt);
    if (state === 'play' || state === 'win' || state === 'code') updateCompanions(dt);
    if (state === 'play' || state === 'code' || state === 'win' || state === 'dead') updateWorld(dt);
    if (state === 'win') updateWin(dt);
    hurtFlash = Math.max(0, hurtFlash - dt * 1.4);
    updateHud();
    const cp = Math.cos(camPitch), target = new THREE.Vector3(P.pos.x + Math.sin(camYaw) * cp * CFG.camDist, P.pos.y + 1.8 + Math.sin(camPitch) * CFG.camDist, P.pos.z + Math.cos(camYaw) * cp * CFG.camDist);
    camera.position.lerp(target, 1 - Math.exp(-12 * dt));
    camera.lookAt(P.pos.x, P.pos.y + 1.8, P.pos.z);
  }
  renderer.render(scene, camera);
}

/* =====================================================================
   11. INPUT + START
   ===================================================================== */
addEventListener('keydown', e => {
  if (e.code === 'KeyM' && state !== 'code') AUDIO.toggle();
  if (state === 'code') { if (e.code === 'Escape') closeCode(); if (e.code === 'Enter') trySolve(); return; }
  if (state !== 'play') return;
  if (['Space', 'ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown'].includes(e.code)) e.preventDefault();
  keys[e.code] = true;
  if (e.code === 'KeyF') wantAttack = true;
  if (e.code === 'KeyE' && !e.repeat) {
    if (!W.solved && !W.boss && Math.hypot(ALTAR.x - P.pos.x, ALTAR.z - P.pos.z) < 6) {
      if (W.collected < CFG.hintsNeeded) toast('The altar is locked. Find all the hints first (' + W.collected + '/' + CFG.hintsNeeded + ').', 4);
      else { e.preventDefault(); openCode(); }
    }
  }
});
addEventListener('keyup', e => { keys[e.code] = false; });
addEventListener('blur', () => { for (const k in keys) keys[k] = false; });
let drag = null;
renderer.domElement.addEventListener('mousedown', e => { drag = { x: e.clientX, y: e.clientY, moved: 0 }; });
addEventListener('mousemove', e => { if (!drag) return; const dx = e.clientX - drag.x, dy = e.clientY - drag.y; drag.x = e.clientX; drag.y = e.clientY; drag.moved += Math.abs(dx) + Math.abs(dy); camYaw += dx * 0.006; camPitch = Math.max(0.12, Math.min(1.15, camPitch + dy * 0.004)); });
addEventListener('mouseup', () => { if (drag && drag.moved < 6 && state === 'play') wantAttack = true; drag = null; });
addEventListener('resize', () => { renderer.setSize(innerWidth, innerHeight); camera.aspect = innerWidth / innerHeight; camera.updateProjectionMatrix(); });
$('codeGo').onclick = trySolve; $('codeCancel').onclick = closeCode;
['d0', 'd1', 'd2'].forEach((id, i, arr) => $(id).addEventListener('input', () => { $(id).value = $(id).value.replace(/[^0-9]/g, ''); if ($(id).value && i < 2) $(arr[i + 1]).focus(); }));

function startGame(key) {
  heroKey = key; hero = HEROES[key];
  if (P.mesh) scene.remove(P.mesh);
  P.mesh = makeHuman(hero.shirt, hero.hair, hero.scale); scene.add(P.mesh);
  comps.forEach(c => scene.remove(c.mesh)); comps.length = 0;
  Object.keys(HEROES).filter(k => k !== key).forEach(k => { const h = HEROES[k], m = makeHuman(h.shirt, h.hair, h.scale); scene.add(m); comps.push({ mesh: m, x: SPAWN.x, z: SPAWN.z, face: 0, walk: 0 }); });
  P.maxHp = hero.hp; P.hp = hero.hp; P.stamina = CFG.staminaMax; deaths = 0;
  $('title').classList.add('hidden'); $('hud').classList.remove('hidden');
  AUDIO.start(); loadDimension(0); camera.position.set(SPAWN.x, 8, SPAWN.z + 10); state = 'play';
  toast('Find the 3 hints. Everything that glows is watching you.', 5);
}
Object.keys(HEROES).forEach(k => {
  const h = HEROES[k], el = document.createElement('div'); el.className = 'hero';
  el.innerHTML = '<b style="color:#' + h.shirt.toString(16).padStart(6, '0') + '">' + h.name + '</b><span>' + h.blurb + '</span>';
  el.onclick = () => startGame(k); $('heroes').appendChild(el);
});
requestAnimationFrame(loop);
})();
</script>
</body>
</html>
