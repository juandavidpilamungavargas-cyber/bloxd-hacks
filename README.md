(function() {
    'use strict';
    if (document.getElementById('wolf-juan-client')) return;

    // ================================================================
    // DETECCIÓN DE MÓVIL
    // ================================================================
    const isMobile = /Android|iPhone|iPad|iPod|webOS/i.test(navigator.userAgent) || ('ontouchstart' in window && window.innerWidth < 1024);
    const SCALE = isMobile ? 0.55 : 1.0;

    // ================================================================
    // CORE (detección robusta, igual que el nexus)
    // ================================================================
    const win = typeof unsafeWindow !== 'undefined' ? unsafeWindow : window;
    const v = (o) => { try { return Object.values(o || {}); } catch(e) { return []; } };
    const a = (f, d = null) => { try { return f(); } catch(e) { return d; } };

    const _mouseState = { left: false };
    window.addEventListener('mousedown', (e) => { if (e.button === 0) _mouseState.left = true; });
    window.addEventListener('mouseup', (e) => { if (e.button === 0) _mouseState.left = false; });

    const G = {
        _n: null,
        get n() {
            if (document.querySelector(".NoConnection")) return null;
            if (this._n) return this._n;
            if (win.noa) { this._n = win.noa; return win.noa; }
            return null;
        },
        get bp() { return this.n ? this.n.bloxd : null; },
        init() {
            if (this._n) return this._n;
            try {
                const elements = [
                    document.querySelector('.LoadingOverlay'),
                    document.querySelector('#root'),
                    document.querySelector('.GameUI')
                ].filter(Boolean);
                for (let i = 0; i < elements.length; i++) {
                    const element = elements[i];
                    const fiberKey = Object.keys(element).find(k => k.startsWith('__reactFiber$'));
                    if (!fiberKey) continue;
                    const fiber = element[fiberKey];
                    const deepFind = (obj, isNoa, seen = new Set()) => {
                        if (!obj || typeof obj !== 'object' || seen.has(obj)) return null;
                        seen.add(obj);
                        try {
                            if (isNoa(obj)) return obj;
                            const values = Object.values(obj);
                            for (let j = 0; j < values.length; j++) {
                                const res = deepFind(values[j], isNoa, seen);
                                if (res) return res;
                            }
                        } catch(e) {}
                        return null;
                    };
                    const isNoa = o => o && o.entities && typeof o.entities.getState === 'function' && o.bloxd && typeof o.bloxd.getPlayerIds === 'function';
                    const found = deepFind(fiber.memoizedProps, isNoa) || deepFind(fiber.memoizedState, isNoa);
                    if (found) { this._n = found; win.noa = found; return found; }
                }
            } catch (e) {}
            return null;
        }
    };

    setInterval(() => { if (!G._n && !win.noa) G.init(); }, 1000);

    const _n2 = (v) => { const s=v[0]*v[0]+v[1]*v[1]+v[2]*v[2]; return s?[v[0]/Math.sqrt(s),v[1]/Math.sqrt(s),v[2]/Math.sqrt(s)]:v; };
    const _d2 = (a,b) => { const dx=b[0]-a[0],dy=b[1]-a[1],dz=b[2]-a[2]; return dx*dx+dy*dy+dz*dz; };
    const ik = () => a(() => { const e = G.n.entities, t = v(e)[2]; return Object.entries(e).find((pair) => pair[1] === t)?.[0]; });

    const getInvItem = () => {
        try {
            const ikey = ik();
            if (!ikey) return null;
            const ent = G.n.entities[ikey];
            const wrapper = Object.values(ent).find(b => b?.list?.[0]?._blockItem);
            return wrapper ? wrapper.list[0] : null;
        } catch { return null; }
    };

    const placeBlock = (pos) => {
        const invItem = getInvItem();
        const blockItem = invItem?._blockItem;
        if (!blockItem?.placeBlock) return;
        const key1 = Object.keys(blockItem)[0];
        const val1 = Object.values(blockItem)[0];
        if (!val1) return;
        const key2 = Object.keys(val1)[25];
        const val2 = val1[key2];
        const proxy = p => new Proxy({}, {
            get: (t, prop) => {
                if (prop === key1) {
                    return new Proxy(val1, {
                        get: (t2, prop2) => prop2 === key2 ? { ...val2, position: p } : val1[prop2]
                    });
                }
                if (prop === 'checkTargetedBlockCanBePlacedOver') return () => true;
                return typeof blockItem[prop] === 'function' ? blockItem[prop].bind(blockItem) : blockItem[prop];
            }
        });
        blockItem.placeBlock.call(proxy(pos));
    };

    const g = {
        p: (id) => a(() => G.n.entities.getState(id, "position").position),
        pl: () => a(() => { const ids = G.bp?.getPlayerIds?.(); return ids ? Object.values(ids).map(Number).filter(id => id !== 1) :[]; },[]),
        hg: () => a(() => v(G.n.entities).find(f => typeof f === "function" && f.length === 1 && f.toString().length < 80 && f.toString().includes(").") && !f.toString().includes("opWrapper")), null),
        sh: (id) => { const getter = g.hg(); return getter ? a(() => getter(id), null) : null; },
        getBlockID: (x, y, z) => { try { const names = Object.getOwnPropertyNames(G.n.constructor.prototype); return G.n[names[4]](x, y, z); } catch { return 0; } },
        at: () => {
            const held = g.sh(1);
            if (!held) return null;
            const target = held.breakingItem || held;
            for (let key in target) {
                if (typeof target[key] === 'function' && target[key].length === 3) return target[key].bind(held);
            }
            const proto = Object.getPrototypeOf(target);
            for (let key of Object.getOwnPropertyNames(proto)) {
                if (typeof target[key] === 'function' && target[key].length === 3) return target[key].bind(held);
            }
            return null;
        },
        sw: () => {
            try {
                const ikey = ik(); if (!ikey) return;
                const ms = G.n.entities?.[ikey]?.moveState?.list?.[0];
                if (ms) ms.isSwinging = true;
                const h = g.sh(1);
                if (h) { if (h.trySwingBlock) h.trySwingBlock(); if (h.trySwingItem) h.trySwingItem(); }
            } catch(e) {}
        },
        c: () => a(() => G.n.camera, null),
        h: (id) => a(() => G.n.entities.getState(id, "health").health, 100)
    };

    // ================================================================
    // TODOS LOS MÓDULOS
    // ================================================================
    const moduleDefinitions = {
        killaura: {
            name: 'Kill Aura', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'Activation', min: 1, max: 3, step: 1, val: 1, display: ['Always','On hit','On hold'] },
                { type: 'range', name: 'Range', min: 3, max: 10, step: 0.1, val: 6.5 },
                { type: 'range', name: 'Delay', min: 0, max: 500, step: 10, val: 50 },
                { type: 'range', name: 'Jab Speed', min: 0, max: 2.0, step: 0.1, val: 0.8 },
                { type: 'range', name: 'Pull Strength', min: -5, max: 5, step: 0.1, val: 0 },
                { type: 'range', name: 'Rotations', min: 0, max: 1, step: 1, val: 1 },
                { type: 'range', name: 'Target HUD', min: 0, max: 1, step: 1, val: 1 },
                { type: 'range', name: 'Multi-Hit', min: 1, max: 5, step: 1, val: 1 }
            ],
            _lastAttack: 0, _jabProgress: 0, _wasLeft: false, _currentTarget: null, _hudElement: null,
            _drawHUD: function(target) {
                const showHUD = this.settings.find(s => s.name === 'Target HUD').val;
                if (!showHUD || !target) {
                    if (this._hudElement) { this._hudElement.style.display = 'none'; }
                    return;
                }
                if (!this._hudElement) {
                    this._hudElement = document.createElement('div');
                    this._hudElement.style = 'position: fixed; top: 65%; left: 50%; transform: translate(-50%, -50%); width: 240px; background: rgba(10, 10, 15, 0.9); border: 1px solid rgba(168, 85, 247, 0.4); border-radius: 12px; padding: 15px; color: white; font-family: "Montserrat", sans-serif; z-index: 1000000; pointer-events: none; backdrop-filter: blur(10px); box-shadow: 0 10px 30px rgba(0,0,0,0.5);';
                    document.body.appendChild(this._hudElement);
                }
                this._hudElement.style.display = 'block';
                const health = g.h(target.id);
                const dist = Math.sqrt(target.dist).toFixed(1);
                this._hudElement.innerHTML = `
                    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
                        <div style="font-weight: 800; color: #a855f7; font-size: 12px; text-transform: uppercase; letter-spacing: 0.05em;">Target Info</div>
                        <div style="font-size: 10px; color: rgba(255,255,255,0.5);">ID: ${target.id}</div>
                    </div>
                    <div style="font-size: 11px; margin-bottom: 8px; color: #e4e4e7;">Distance: <span style="color: #a855f7; font-weight: bold;">${dist}m</span></div>
                    <div style="width: 100%; height: 6px; background: rgba(255,255,255,0.05); border-radius: 3px; overflow: hidden; margin-bottom: 4px;">
                        <div style="width: ${health}%; height: 100%; background: linear-gradient(90deg, #a855f7, #6366f1); transition: width 0.4s cubic-bezier(0.4, 0, 0.2, 1);"></div>
                    </div>
                    <div style="display: flex; justify-content: space-between; font-size: 10px; font-weight: 600;">
                        <span style="color: rgba(255,255,255,0.4);">Health</span>
                        <span style="color: #a855f7;">${health.toFixed(1)}%</span>
                    </div>
                `;
            },
            onTick: function() {
                const activation = this.settings.find(s => s.name === 'Activation').val;
                if (activation === 3 && !_mouseState.left) return;
                if (activation === 2) {
                    const isClick = _mouseState.left && !this._wasLeft;
                    this._wasLeft = _mouseState.left;
                    if (!isClick) return;
                }
                this._wasLeft = _mouseState.left;
                const now = Date.now();
                const range = this.settings.find(s => s.name === 'Range').val;
                const delay = this.settings.find(s => s.name === 'Delay').val;
                const jabSpeed = this.settings.find(s => s.name === 'Jab Speed').val;
                const pull = this.settings.find(s => s.name === 'Pull Strength').val;
                const item = g.sh(1);
                if (item) {
                    const target = item._standardItem || item;
                    if (jabSpeed > 0) {
                        this._jabProgress += jabSpeed;
                        if (this._jabProgress > Math.PI * 2) this._jabProgress -= Math.PI * 2;
                        if (target.firstPersonPosOffset) {
                            const thrust = Math.sin(this._jabProgress) * 0.8;
                            target.firstPersonPosOffset.z = thrust;
                            target.firstPersonPosOffset.x = -thrust * 0.2;
                        }
                        if (target.firstPersonRotation) {
                            target.firstPersonRotation.x = -Math.sin(this._jabProgress) * 0.3;
                        }
                    } else {
                        if (target.firstPersonPosOffset) { target.firstPersonPosOffset.z = 0; target.firstPersonPosOffset.x = 0; }
                        if (target.firstPersonRotation) { target.firstPersonRotation.x = 0; }
                        this._jabProgress = 0;
                    }
                }
                if (now - this._lastAttack < delay) return;
                const pp = g.p(1); if (!pp) return;
                const rangeSq = range * range;
                let targets = g.pl().map(id => {
                    const pos = g.p(id);
                    return { id, pos, dist: pos ? _d2(pp, pos) : Infinity };
                }).filter(t => t.pos && t.dist <= rangeSq);
                if (targets.length === 0) { this._currentTarget = null; this._drawHUD(null); return; }
                targets.sort((a, b) => a.dist - b.dist);
                this._currentTarget = targets[0];
                this._drawHUD(this._currentTarget);
                const atFn = g.at();
                if (atFn) {
                    const rotations = this.settings.find(s => s.name === 'Rotations').val;
                    const multiHit = this.settings.find(s => s.name === 'Multi-Hit').val;
                    targets.forEach(target => {
                        let vec = _n2([target.pos[0] - pp[0], target.pos[1] - pp[1], target.pos[2] - pp[2]]);
                        if (pull < 0) { vec[0] = -vec[0]; vec[1] = -vec[1]; vec[2] = -vec[2]; }
                        if (rotations === 1 && G.n && G.n.camera) {
                            const dx = target.pos[0] - pp[0], dy = target.pos[1] - pp[1], dz = target.pos[2] - pp[2];
                            const dist = Math.sqrt(dx*dx + dy*dy + dz*dz);
                            G.n.camera.heading = Math.atan2(dx, dz);
                            G.n.camera.pitch = -Math.asin(dy / dist);
                        }
                        for (let i = 0; i < multiHit; i++) { atFn(vec, target.id.toString(), 'BodyMesh'); }
                        g.sw();
                    });
                    this._lastAttack = now;
                }
            },
            onChange: function(enabled) {
                if (!enabled) {
                    if (this._hudElement) { this._hudElement.style.display = 'none'; }
                    const item = g.sh(1);
                    if (item) {
                        const target = item._standardItem || item;
                        if (target.firstPersonPosOffset) { target.firstPersonPosOffset.x = 0; target.firstPersonPosOffset.z = 0; }
                        if (target.firstPersonRotation) { target.firstPersonRotation.x = 0; }
                    }
                }
            }
        },
        spinbot: {
            name: 'SpinBot', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'Speed', min: 0.1, max: 2.0, step: 0.1, val: 0.5 }
            ],
            _yaw: 0,
            onTick: function() {
                const speed = this.settings.find(s => s.name === 'Speed').val;
                this._yaw += speed;
                if (G.n?.camera) G.n.camera.heading = this._yaw;
            }
        },
        triggerbot: {
            name: 'Trigger Bot', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'FOV', min: 1, max: 30, step: 0.5, val: 5 },
                { type: 'range', name: 'Reaction Delay', min: 0, max: 300, step: 10, val: 50 }
            ],
            _lastShot: 0,
            onTick: function() {
                const now = Date.now();
                const delay = this.settings.find(s => s.name === 'Reaction Delay').val;
                const fov = this.settings.find(s => s.name === 'FOV').val;
                if (now - this._lastShot < delay) return;
                const p = g.p(1); const c = g.c(); if (!p || !c) return;
                const playerIds = g.pl();
                for (let i = 0; i < playerIds.length; i++) {
                    const id = playerIds[i];
                    const ep = g.p(id); if (!ep) continue;
                    const dx = ep[0] - p[0], dy = ep[1] - p[1], dz = ep[2] - p[2];
                    const dist = Math.sqrt(dx*dx + dy*dy + dz*dz); if (dist > 50) continue;
                    const targetYaw = Math.atan2(dx, dz), targetPitch = -Math.asin(dy / dist);
                    let dyaw = targetYaw - c.heading;
                    while (dyaw > Math.PI) dyaw -= 2 * Math.PI; while (dyaw < -Math.PI) dyaw += 2 * Math.PI;
                    const dpitch = targetPitch - c.pitch;
                    const diff = Math.sqrt(dyaw*dyaw + dpitch*dpitch) * (180 / Math.PI);
                    if (diff <= fov) {
                        if (G.n?.inputs?.down?.emit) {
                            G.n.inputs.down.emit('primary-fire', true);
                            setTimeout(() => { if (G.n?.inputs?.up?.emit) G.n.inputs.up.emit('primary-fire', true); }, 10);
                            g.sw(); this._lastShot = now; break;
                        }
                    }
                }
            }
        },
        autoclicker: {
            name: 'Auto Clicker', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'Min CPS', min: 1, max: 20, step: 1, val: 10 },
                { type: 'range', name: 'Max CPS', min: 1, max: 20, step: 1, val: 14 }
            ],
            _lastClick: 0,
            onTick: function() {
                if (!_mouseState.left) return;
                const minCps = this.settings.find(s=>s.name==='Min CPS').val;
                const maxCps = this.settings.find(s=>s.name==='Max CPS').val;
                const cps = minCps + (Math.random() * (maxCps - minCps));
                const now = Date.now();
                if (now - this._lastClick < 1000 / cps) return;
                try {
                    if (G.n?.inputs?.down?.emit) {
                        G.n.inputs.down.emit('primary-fire', true);
                        setTimeout(() => { if (G.n?.inputs?.up?.emit) G.n.inputs.up.emit('primary-fire', true); }, 10);
                    }
                    g.sw();
                } catch(e) {}
                this._lastClick = now;
            }
        },
        reach: {
            name: 'Reach', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'Distance', min: 3, max: 8, step: 0.1, val: 4.5 }
            ],
            _originalKillAuraRange: null,
            onTick: function() {
                if (!this._originalKillAuraRange) {
                    const killAuraModule = Hacks.killaura;
                    if (killAuraModule) {
                        const rangeSetting = killAuraModule.settings.find(s => s.name === 'Range');
                        if (rangeSetting) { this._originalKillAuraRange = rangeSetting.val; }
                    }
                }
                const distance = this.settings.find(s => s.name === 'Distance').val;
                const killAuraModule = Hacks.killaura;
                if (killAuraModule) {
                    const rangeSetting = killAuraModule.settings.find(s => s.name === 'Range');
                    if (rangeSetting) { rangeSetting.val = distance; }
                }
            },
            onChange: function(enabled) {
                if (!enabled && this._originalKillAuraRange !== null) {
                    const killAuraModule = Hacks.killaura;
                    if (killAuraModule) {
                        const rangeSetting = killAuraModule.settings.find(s => s.name === 'Range');
                        if (rangeSetting) { rangeSetting.val = this._originalKillAuraRange; this._originalKillAuraRange = null; }
                    }
                }
            }
        },
        swinganim: {
            name: 'Swing Animation', category: 'Combat', hasSettings: true,
            settings: [
                { type: 'range', name: 'Style', min: 1, max: 3, step: 1, val: 1, display: ['Wave','Spin','Stab'] },
                { type: 'range', name: 'Speed', min: 0.1, max: 2.0, step: 0.1, val: 1.0 }
            ],
            _progress: 0,
            onTick: function() {
                const killAura = Hacks.killaura;
                if (!killAura || !killAura.enabled) return;
                const item = g.sh(1); if (!item) return;
                const target = item._standardItem || item;
                const style = this.settings.find(s => s.name === 'Style').val;
                const speed = this.settings.find(s => s.name === 'Speed').val;
                this._progress += speed * 0.2;
                if (this._progress > Math.PI * 2) this._progress -= Math.PI * 2;
                if (target.firstPersonPosOffset && target.firstPersonRotation) {
                    if (style === 1) {
                        target.firstPersonPosOffset.z = Math.sin(this._progress) * 0.5;
                        target.firstPersonRotation.x = -Math.cos(this._progress) * 0.5;
                    } else if (style === 2) {
                        target.firstPersonRotation.y = this._progress;
                        target.firstPersonPosOffset.x = Math.sin(this._progress) * 0.3;
                    } else if (style === 3) {
                        target.firstPersonPosOffset.z = Math.abs(Math.sin(this._progress)) * -1.2;
                        target.firstPersonRotation.x = 0.2;
                    }
                }
            },
            onChange: function(enabled) {
                if (!enabled) {
                    const item = g.sh(1);
                    if (item) {
                        const target = item._standardItem || item;
                        if (target.firstPersonPosOffset) { target.firstPersonPosOffset.x = 0; target.firstPersonPosOffset.z = 0; }
                        if (target.firstPersonRotation) { target.firstPersonRotation.x = 0; target.firstPersonRotation.y = 0; }
                    }
                }
            }
        },
        autojump: {
            name: 'Auto Jump', category: 'Movement', hasSettings: false,
            settings: [],
            onTick: function() { if (G.n?.inputs?.state) G.n.inputs.state.jump = 1; },
            onChange: function(enabled) { if (!enabled && G.n?.inputs?.state) G.n.inputs.state.jump = 0; }
        },
        bhop: {
            name: 'Bhop', category: 'Movement', hasSettings: true,
            settings: [
                { type: 'range', name: 'Jump Delay', min: 0, max: 200, step: 10, val: 20 }
            ],
            _lastJump: 0,
            onTick: function() {
                try {
                    const ikey = ik(); if (!ikey) return;
                    const mv = G.n.entities?.[ikey]?.movement?.list?.[0];
                    if (!mv) return;
                    const og = typeof mv.isOnGround === 'function' ? mv.isOnGround() : (mv.onGroundPrevTick || !mv.isInAir);
                    const now = Date.now();
                    const delay = this.settings.find(s => s.name === 'Jump Delay').val;
                    if (og && (now - this._lastJump >= delay)) {
                        if (G.n?.inputs?.state) {
                            G.n.inputs.state.jump = 1;
                            if (mv._hadJumpInputPrevTick !== undefined) mv._hadJumpInputPrevTick = false;
                            if (mv.isInAir !== undefined) mv.isInAir = false;
                        }
                        this._lastJump = now;
                    } else {
                        if (G.n?.inputs?.state && !og) {
                            G.n.inputs.state.jump = 0;
                        }
                    }
                } catch(e) {}
            }
        },
        autosprint: {
            name: 'Auto Sprint', category: 'Movement', hasSettings: false,
            settings: [],
            onTick: function() {
                if (G.n?.inputs?.state) {
                    if (G.n.inputs.state.forward) { G.n.inputs.state.sprint = 1; }
                    else { G.n.inputs.state.sprint = 0; }
                }
            },
            onChange: function(enabled) { if (!enabled && G.n?.inputs?.state) { G.n.inputs.state.sprint = 0; } }
        },
        scaffold: {
            name: 'Scaffold', category: 'Movement', hasSettings: true,
            settings: [
                { type: 'toggle', name: 'Lock Y', val: false },
                { type: 'toggle', name: 'AutoJump', val: false }
            ],
            _isBuilding: false, _lockY: null,
            onChange: function(enabled) { if (!enabled) { this._lockY = null; } },
            onTick: function() {
                if (this._isBuilding) return;
                const invItem = getInvItem();
                if (invItem?.heldType !== 'CubeBlock') { this._lockY = null; return; }
                this._isBuilding = true;
                const pos = g.p(1);
                if (pos) {
                    const lockYSetting = this.settings.find(s=>s.name==='Lock Y').val;
                    const autoJump = this.settings.find(s=>s.name==='AutoJump').val;
                    let targetY = Math.floor(pos[1] - 1);
                    if (lockYSetting) {
                        if (this._lockY === null) this._lockY = targetY;
                        targetY = this._lockY;
                    } else { this._lockY = null; }
                    const state = G.n?.inputs?.state;
                    const ikey = ik();
                    if (autoJump && state && ikey) {
                        const mv = G.n.entities?.[ikey]?.movement?.list?.[0];
                        const onGround = mv?.isOnGround?.() ?? false;
                        const isMoving = state.forward || state.backward || state.left || state.right;
                        if (isMoving && onGround) { state.jump = 1; }
                    }
                    const targetPos = [Math.floor(pos[0]), targetY, Math.floor(pos[2])];
                    if (g.getBlockID(targetPos[0], targetPos[1], targetPos[2]) === 0) { placeBlock(targetPos); }
                }
                this._isBuilding = false;
            }
        },
        swordspin: {
            name: 'Sword Spin', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Speed', min: 0.01, max: 0.5, step: 0.01, val: 0.1 }
            ],
            _angle: 0,
            onTick: function() {
                const item = g.sh(1); if (!item) return;
                const target = item._standardItem || item;
                const speed = this.settings.find(s => s.name === 'Speed').val;
                this._angle += speed;
                if (target.firstPersonRotation) { target.firstPersonRotation.y = this._angle; target.firstPersonRotation.x = this._angle * 0.5; }
            },
            onChange: function(enabled) {
                if (!enabled) {
                    const item = g.sh(1);
                    if (item) {
                        const target = item._standardItem || item;
                        if (target.firstPersonRotation) { target.firstPersonRotation.x = 0; target.firstPersonRotation.y = 0; }
                    }
                }
            }
        },
        nametags: {
            name: 'Name Tags', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Scale', min: 1, max: 5, step: 0.1, val: 2.0 }
            ],
            _canvas: null, _ctx: null,
            onTick: function() {
                if (!this._canvas) {
                    this._canvas = document.createElement('canvas');
                    this._canvas.id = 'nexus-nametags-canvas';
                    this._canvas.style.position = 'fixed'; this._canvas.style.top = '0'; this._canvas.style.left = '0';
                    this._canvas.style.width = '100%'; this._canvas.style.height = '100%';
                    this._canvas.style.pointerEvents = 'none'; this._canvas.style.zIndex = '999998';
                    document.body.appendChild(this._canvas);
                    this._ctx = this._canvas.getContext('2d');
                }
                this._canvas.width = window.innerWidth; this._canvas.height = window.innerHeight;
                this._ctx.clearRect(0, 0, this._canvas.width, this._canvas.height);
                const scale = this.settings.find(s => s.name === 'Scale').val;
                const p = g.p(1); const c = g.c(); if (!p || !c) return;
                const playerIds = g.pl();
                for (let i = 0; i < playerIds.length; i++) {
                    const id = playerIds[i];
                    const ep = g.p(id); if (!ep) continue;
                    const dx = ep[0] - p[0], dy = ep[1] - p[1] + 2, dz = ep[2] - p[2];
                    const cosY = Math.cos(-c.heading), sinY = Math.sin(-c.heading);
                    const cosX = Math.cos(-c.pitch), sinX = Math.sin(-c.pitch);
                    let rx = dx * cosY - dz * sinY, rz = dx * sinY + dz * cosY, ry = dy * cosX - rz * sinX; rz = dy * sinX + rz * cosX;
                    if (rz > 0.1) {
                        const fov = 1000;
                        const screenX = (rx / rz) * fov + window.innerWidth / 2;
                        const screenY = -(ry / rz) * fov + window.innerHeight / 2;
                        if (screenX > 0 && screenX < window.innerWidth && screenY > 0 && screenY < window.innerHeight) {
                            this._ctx.font = "bold " + (12 * scale) + "px 'Montserrat', sans-serif";
                            this._ctx.fillStyle = 'white'; this._ctx.textAlign = 'center';
                            this._ctx.shadowBlur = 4; this._ctx.shadowColor = 'black';
                            this._ctx.fillText("Player " + id, screenX, screenY);
                        }
                    }
                }
            },
            onChange: function(enabled) { if (!enabled && this._canvas) { this._canvas.remove(); this._canvas = null; } }
        },
        fullbright: {
            name: 'Fullbright', category: 'Visual', hasSettings: false,
            settings: [],
            onTick: function() {
                try {
                    if (G.n?.rendering?._scene) G.n.rendering._scene.ambientColor = { r: 1, g: 1, b: 1 };
                    document.body.style.filter = 'brightness(1.5)';
                } catch(e) {}
            },
            onChange: function(enabled) {
                if (!enabled) {
                    try {
                        if (G.n?.rendering?._scene) G.n.rendering._scene.ambientColor = { r: 0.5, g: 0.5, b: 0.5 };
                        document.body.style.filter = '';
                    } catch(e) {}
                }
            }
        },
        reversehead: {
            name: 'Reverse Head', category: 'Visual', hasSettings: false,
            settings: [],
            _originalHeading: null,
            onTick: function() {
                if (G.n?.camera) {
                    if (this._originalHeading === null) { this._originalHeading = G.n.camera.heading; }
                    G.n.camera.heading = this._originalHeading + Math.PI;
                }
            },
            onChange: function(enabled) {
                if (!enabled && G.n?.camera && this._originalHeading !== null) {
                    G.n.camera.heading = this._originalHeading;
                    this._originalHeading = null;
                }
            }
        },
        headdown: {
            name: 'Head Down', category: 'Visual', hasSettings: false,
            settings: [],
            _originalPitch: null,
            onTick: function() {
                if (G.n?.camera) {
                    if (this._originalPitch === null) { this._originalPitch = G.n.camera.pitch; }
                    G.n.camera.pitch = Math.PI / 2;
                }
            },
            onChange: function(enabled) {
                if (!enabled && G.n?.camera && this._originalPitch !== null) {
                    G.n.camera.pitch = this._originalPitch;
                    this._originalPitch = null;
                }
            }
        },
        tracers: {
            name: 'Tracers', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Width', min: 1, max: 5, step: 0.5, val: 1.5 }
            ],
            _canvas: null, _ctx: null,
            onTick: function() {
                if (!this._canvas) {
                    this._canvas = document.createElement('canvas');
                    this._canvas.id = 'nexus-tracers-canvas';
                    this._canvas.style.position = 'fixed'; this._canvas.style.top = '0'; this._canvas.style.left = '0';
                    this._canvas.style.width = '100%'; this._canvas.style.height = '100%';
                    this._canvas.style.pointerEvents = 'none'; this._canvas.style.zIndex = '999998';
                    document.body.appendChild(this._canvas);
                    this._ctx = this._canvas.getContext('2d');
                }
                this._canvas.width = window.innerWidth; this._canvas.height = window.innerHeight;
                this._ctx.clearRect(0, 0, this._canvas.width, this._canvas.height);
                const width = this.settings.find(s => s.name === 'Width').val;
                const p = g.p(1); const c = g.c(); if (!p || !c) return;
                const playerIds = g.pl();
                for (let i = 0; i < playerIds.length; i++) {
                    const id = playerIds[i];
                    const ep = g.p(id); if (!ep) continue;
                    const dx = ep[0] - p[0], dy = ep[1] - p[1] + 1, dz = ep[2] - p[2];
                    const cosY = Math.cos(-c.heading), sinY = Math.sin(-c.heading);
                    const cosX = Math.cos(-c.pitch), sinX = Math.sin(-c.pitch);
                    let rx = dx * cosY - dz * sinY, rz = dx * sinY + dz * cosY, ry = dy * cosX - rz * sinX; rz = dy * sinX + rz * cosX;
                    if (rz > 0.1) {
                        const fov = 1000;
                        const screenX = (rx / rz) * fov + window.innerWidth / 2;
                        const screenY = -(ry / rz) * fov + window.innerHeight / 2;
                        if (screenX > 0 && screenX < window.innerWidth && screenY > 0 && screenY < window.innerHeight) {
                            this._ctx.beginPath();
                            this._ctx.moveTo(window.innerWidth / 2, window.innerHeight);
                            this._ctx.lineTo(screenX, screenY);
                            this._ctx.lineWidth = width;
                            this._ctx.strokeStyle = 'rgba(255, 0, 0, 0.7)';
                            this._ctx.stroke();
                        }
                    }
                }
            },
            onChange: function(enabled) { if (!enabled && this._canvas) { this._canvas.remove(); this._canvas = null; } }
        },
        custompose: {
            name: 'Custom Pose', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Pose ID', min: 0, max: 5, step: 1, val: 0, display: ['standing','zombie','sitting','lying','sneaking','swimming'] }
            ],
            _poses: ["standing", "zombie", "sitting", "lying", "sneaking", "swimming"],
            onTick: function() {
                const ikey = ik(); if (!ikey) return;
                const ms = a(() => G.n.entities?.[ikey]?.moveState?.list?.[0], null);
                const poseIdx = Math.floor(this.settings.find(s => s.name === 'Pose ID').val);
                const pose = this._poses[poseIdx] || "standing";
                if (ms?.setPose) ms.setPose(pose);
            },
            onChange: function(enabled) {
                if (!enabled) {
                    const ikey = ik(); if (!ikey) return;
                    const ms = a(() => G.n.entities?.[ikey]?.moveState?.list?.[0], null);
                    if (ms?.setPose) ms.setPose("standing");
                }
            }
        },
        viewmodel: {
            name: 'ViewModel', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Pos X', min: -3, max: 3, step: 0.05, val: 0 },
                { type: 'range', name: 'Pos Y', min: -3, max: 3, step: 0.05, val: 0 },
                { type: 'range', name: 'Pos Z', min: -3, max: 3, step: 0.05, val: 0 },
                { type: 'range', name: 'Rot X', min: -3.14, max: 3.14, step: 0.01, val: 0 },
                { type: 'range', name: 'Rot Y', min: -3.14, max: 3.14, step: 0.01, val: 0 },
                { type: 'range', name: 'Rot Z', min: -3.14, max: 3.14, step: 0.01, val: 0 }
            ],
            onTick: function() {
                const item = g.sh(1); if (!item) return;
                const target = item._standardItem || item;
                const px = this.settings.find(s=>s.name==='Pos X').val, py = this.settings.find(s=>s.name==='Pos Y').val, pz = this.settings.find(s=>s.name==='Pos Z').val;
                const rx = this.settings.find(s=>s.name==='Rot X').val, ry = this.settings.find(s=>s.name==='Rot Y').val, rz = this.settings.find(s=>s.name==='Rot Z').val;
                if (target.firstPersonPosOffset) { target.firstPersonPosOffset.x = px; target.firstPersonPosOffset.y = py; target.firstPersonPosOffset.z = pz; }
                if (target.firstPersonRotation) { target.firstPersonRotation.x = rx; target.firstPersonRotation.y = ry; target.firstPersonRotation.z = rz; }
            },
            onChange: function(enabled) {
                if (!enabled) {
                    const item = g.sh(1);
                    if (item) {
                        const target = item._standardItem || item;
                        if (target.firstPersonPosOffset) { target.firstPersonPosOffset.x = 0; target.firstPersonPosOffset.y = 0; target.firstPersonPosOffset.z = 0; }
                        if (target.firstPersonRotation) { target.firstPersonRotation.x = 0; target.firstPersonRotation.y = 0; target.firstPersonRotation.z = 0; }
                    }
                }
            }
        },
        characteranims: {
            name: 'Character Anims', category: 'Visual', hasSettings: true,
            settings: [
                { type: 'range', name: 'Mode', min: 1, max: 4, step: 1, val: 1, display: ['Twerk','Headless','Drunk','Pose Cycle'] },
                { type: 'range', name: 'Speed', min: 0.1, max: 2.0, step: 0.1, val: 1.0 }
            ],
            _timer: 0,
            onTick: function() {
                const ikey = ik(); if (!ikey) return;
                const ms = a(() => G.n.entities?.[ikey]?.moveState?.list?.[0], null);
                if (!ms) return;
                const mode = this.settings.find(s => s.name === 'Mode').val;
                const speed = this.settings.find(s => s.name === 'Speed').val;
                this._timer += speed * 0.1;
                if (mode === 1) {
                    if (G.n?.inputs?.state) G.n.inputs.state.sneak = Math.sin(this._timer * 5) > 0 ? 1 : 0;
                } else if (mode === 2) {
                    if (ms.setPose) ms.setPose("lying");
                } else if (mode === 3) {
                    if (ms.setPose) {
                        const drunkPoses = ["standing", "sneaking"];
                        const idx = Math.floor(this._timer % drunkPoses.length);
                        ms.setPose(drunkPoses[idx]);
                    }
                } else if (mode === 4) {
                    const poses = ["standing", "zombie", "sitting", "lying", "sneaking", "swimming"];
                    const idx = Math.floor(this._timer % poses.length);
                    if (ms.setPose) ms.setPose(poses[idx]);
                }
            },
            onChange: function(enabled) {
                if (!enabled) {
                    const ikey = ik(); if (!ikey) return;
                    const ms = a(() => G.n.entities?.[ikey]?.moveState?.list?.[0], null);
                    if (ms?.setPose) ms.setPose("standing");
                    if (G.n?.inputs?.state) G.n.inputs.state.sneak = 0;
                }
            }
        },
        arraylist: {
            name: 'Arraylist', category: 'Visual', hasSettings: false,
            settings: [],
            onTick: function() {}
        },
        chams: {
            name: 'Chams', category: 'Visual', hasSettings: false,
            settings: [],
            _applyChams: function(enable) {
                if (!G.n) return;
                try {
                    let rendering = v(G.n)[12]; if (!rendering) return;
                    let thinMeshes = v(rendering).find(value => value?.thinMeshes)?.thinMeshes;
                    if (!Array.isArray(thinMeshes)) return;
                    let renderingGroupId = enable ? 2 : 0;
                    for (let i = 0; i < thinMeshes.length; i++) {
                        let item = thinMeshes[i];
                        let mesh = item?.meshVariations?.__DEFAULT__?.mesh;
                        if (mesh && typeof mesh.renderingGroupId === "number") mesh.renderingGroupId = renderingGroupId;
                    }
                } catch(e) {}
            },
            onChange: function(enabled) { this._applyChams(enabled); },
            onTick: function() { this._applyChams(true); }
        },
        newaccount: {
            name: 'New Account', category: 'Misc', hasSettings: false,
            settings: [],
            onChange: function(enabled) {
                if (enabled) {
                    // Mostrar pantalla de carga falsa para simular un bypass limpio
                    const blocker = document.createElement('div');
                    blocker.style = "position:fixed;top:0;left:0;width:100%;height:100%;background:#0a0a0f;color:#a855f7;z-index:9999999;display:flex;flex-direction:column;align-items:center;justify-content:center;font-family:sans-serif;font-size:24px;font-weight:bold;";
                    blocker.innerHTML = "<div>🐺 WOLF JUAN CLIENT</div><div style='font-size:14px;color:#fff;margin-top:10px;'>Bypassing Ban Evasion & Purging Identifiers...</div>";
                    document.body.appendChild(blocker);

                    // 1. PURGA ULTRA DE COOKIES (Borrando perfiles en dominios cruzados)
                    const cookies = document.cookie.split(";");
                    for (let i = 0; i < cookies.length; i++) {
                        const name = cookies[i].split("=")[0].trim();
                        document.cookie = name + "=;expires=Thu, 01 Jan 1970 00:00:00 GMT;path=/";
                        document.cookie = name + "=;expires=Thu, 01 Jan 1970 00:00:00 GMT;path=/;domain=" + window.location.hostname;
                        document.cookie = name + "=;expires=Thu, 01 Jan 1970 00:00:00 GMT;path=/;domain=.bloxd.io";
                    }

                    // 2. DESTRUCCIÓN ABSOLUTA DE LOCAL STORAGE (Elimina ID Player y configuraciones de cuenta)
                    try {
                        localStorage.clear();
                        Object.keys(localStorage).forEach(k => localStorage.removeItem(k));
                    } catch(e) {}

                    // 3. DESTRUCCIÓN ABSOLUTA DE SESSION STORAGE (Borrando vms--user_session_id)
                    try {
                        sessionStorage.clear();
                        Object.keys(sessionStorage).forEach(k => sessionStorage.removeItem(k));
                    } catch(e) {}

                    // 4. ANHILACIÓN DE INDEXEDDB (Donde Bloxd guarda perfiles, logs de baneos y datos persistentes del juego)
                    try {
                        if (window.indexedDB && typeof window.indexedDB.databases === 'function') {
                            window.indexedDB.databases().then(dbs => {
                                dbs.forEach(db => {
                                    if(db.name) window.indexedDB.deleteDatabase(db.name);
                                });
                            });
                        }
                        // Borrado forzado de bases de datos comunes del motor del juego
                        window.indexedDB.deleteDatabase("BloxdDatabase");
                        window.indexedDB.deleteDatabase("localforage");
                        window.indexedDB.deleteDatabase("firebaseLocalStorageDb");
                    } catch(e) {}

                    // 5. CACHÉ STORAGE PURGE (Evita el tracking por service workers modificados)
                    try {
                        if (window.caches) {
                            window.caches.keys().then(keys => {
                                keys.forEach(key => window.caches.delete(key));
                            });
                        }
                    } catch(e) {}

                    // 6. DESASOCIAR SERVICE WORKERS ACTIVOS
                    try {
                        if (navigator.serviceWorker) {
                            navigator.serviceWorker.getRegistrations().then(regs => {
                                regs.forEach(reg => reg.unregister());
                            });
                        }
                    } catch(e) {}

                    // 7. EJECUTAR REINICIO TOTAL TRAS LIMPIEZA
                    setTimeout(() => {
                        window.location.replace(window.location.origin + "?nocache=" + Date.now());
                    }, 1200);
                }
            }
        }
    };

    // Inicializar Hacks
    const Hacks = {};
    Object.keys(moduleDefinitions).forEach(key => {
        const def = moduleDefinitions[key];
        Hacks[key] = {
            name: def.name,
            enabled: false,
            keybind: 'None',
            category: def.category,
            hasSettings: def.hasSettings,
            settings: def.settings ? def.settings.map(s => ({...s})) : [],
            onTick: def.onTick,
            onChange: def.onChange,
            ...Object.fromEntries(Object.entries(def).filter(([k]) => !['name','category','hasSettings','settings','onTick','onChange'].includes(k)))
        };
    });

    // ================================================================
    // HUD
    // ================================================================
    const HUD = {
        el: null, items: new Map(),
        init() {
            this.el = document.createElement('div');
            this.el.style.cssText = `position:fixed;top:12px;right:12px;display:flex;flex-direction:column;gap:3px;z-index:999999;pointer-events:none;transform:scale(${SCALE});transform-origin:top right;`;
            document.body.appendChild(this.el);
        },
        update() {
            const active = [];
            for (const [k, h] of Object.entries(Hacks)) { if (h.enabled && h.name) active.push(h.name); }
            for (const [n, it] of this.items.entries()) { if (!active.includes(n)) { it.el.remove(); this.items.delete(n); } }
            for (const n of active) {
                if (!this.items.has(n)) {
                    const el = document.createElement('div'); el.textContent = n;
                    el.style.cssText = `background:rgba(0,0,0,0.65);color:#fff;font-size:10px;font-weight:700;font-family:'Segoe UI',Arial,sans-serif;padding:3px 8px;border-radius:4px;backdrop-filter:blur(4px);white-space:nowrap;border:1.5px solid transparent;border-image:linear-gradient(90deg,#00BFFF,#FFD700,#FF4444)1;animation:wjc-hud 2s linear infinite;letter-spacing:0.5px;`;
                    this.el.appendChild(el); this.items.set(n, { el });
                }
            }
        }
    };
    const hudStyle = document.createElement('style');
    hudStyle.textContent = `@keyframes wjc-hud{0%{border-image:linear-gradient(90deg,#00BFFF,#FFD700,#FF4444)1}33%{border-image:linear-gradient(90deg,#FFD700,#FF4444,#00BFFF)1}66%{border-image:linear-gradient(90deg,#FF4444,#00BFFF,#FFD700)1}100%{border-image:linear-gradient(90deg,#00BFFF,#FFD700,#FF4444)1}}`;
    document.head.appendChild(hudStyle);

    // ================================================================
    // BOTÓN DEL LOBO
    // ================================================================
    const wolfBtn = document.createElement('button');
    wolfBtn.textContent = '🐺';
    wolfBtn.style.cssText = `
        position: fixed; bottom: ${20*SCALE}px; right: ${20*SCALE}px;
        width: ${52*SCALE}px; height: ${52*SCALE}px; border-radius: 50%;
        background: rgba(0,0,0,0.7); border: 2px solid rgba(255,255,255,0.3);
        font-size: ${28*SCALE}px; cursor: pointer; z-index: 1000000;
        backdrop-filter: blur(6px); display: flex; align-items: center; justify-content: center;
        user-select: none;
    `;
    wolfBtn.addEventListener('mouseenter', () => { wolfBtn.style.transform = 'scale(1.1)'; wolfBtn.style.borderColor = 'rgba(255,255,255,0.6)'; });
    wolfBtn.addEventListener('mouseleave', () => { wolfBtn.style.transform = ''; wolfBtn.style.borderColor = 'rgba(255,255,255,0.3)'; });
    document.body.appendChild(wolfBtn);

    let wolfDragging = false, wolfStartX, wolfStartY, wolfOffsetX = 0, wolfOffsetY = 0;
    let wasDragged = false;

    wolfBtn.addEventListener('mousedown', (e) => {
        if (e.button === 0) {
            wolfDragging = true;
            wasDragged = false;
            wolfStartX = e.clientX - wolfOffsetX;
            wolfStartY = e.clientY - wolfOffsetY;
            wolfBtn.style.cursor = 'grabbing';
            e.preventDefault();
        }
    });

    document.addEventListener('mousemove', (e) => {
        if (wolfDragging) {
            const newX = e.clientX - wolfStartX;
            const newY = e.clientY - wolfStartY;
            if (Math.abs(newX - wolfOffsetX) > 2 || Math.abs(newY - wolfOffsetY) > 2) {
                wasDragged = true;
            }
            wolfOffsetX = newX;
            wolfOffsetY = newY;
            wolfBtn.style.right = 'auto';
            wolfBtn.style.bottom = 'auto';
            wolfBtn.style.left = (window.innerWidth + wolfOffsetX - 52*SCALE) + 'px';
            wolfBtn.style.top = (window.innerHeight + wolfOffsetY - 52*SCALE) + 'px';
        }
    });

    document.addEventListener('mouseup', (e) => {
        if (wolfDragging) {
            wolfDragging = false;
            wolfBtn.style.cursor = 'pointer';
            if (!wasDragged) {
                toggleMenu();
            }
        }
    });

    // ================================================================
    // INTERFAZ DEL MENÚ
    // ================================================================
    const overlay = document.createElement('div'); overlay.id = 'wjc-overlay';
    overlay.style.cssText = `position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.5);z-index:999998;display:none;pointer-events:none;`;
    document.body.appendChild(overlay);

    const title = document.createElement('div'); title.id = 'wjc-title'; title.textContent = 'Wolf Juan Client';
    title.style.cssText = `position:fixed;top:10px;left:10px;z-index:999999;font-family:'Segoe UI',Arial,sans-serif;font-size:${32*SCALE}px;font-weight:bold;text-shadow:2px 2px 4px rgba(0,0,0,0.5);letter-spacing:1px;user-select:none;cursor:default;display:block;pointer-events:none;`;
    document.body.appendChild(title);

    const container = document.createElement('div'); container.id = 'wolf-juan-client';
    container.style.cssText = `position:fixed;top:${60*SCALE}px;left:${10*SCALE}px;z-index:999999;font-family:'Segoe UI',Arial,sans-serif;user-select:none;display:none;transition:all 0.4s cubic-bezier(0.68,-0.55,0.265,1.55);transform:perspective(1000px) rotateX(-90deg);transform-origin:top center;opacity:0;`;

    const catCont = document.createElement('div');
    catCont.style.cssText = `display:flex;flex-direction:row;gap:${40*SCALE}px;align-items:flex-start;`;

    const categories = ['Combat','Movement','Visual','Misc'];
    const bp = {};

    const ch = {
        Combat: [
            {k:'killaura',s:1},{k:'spinbot',s:1},{k:'triggerbot',s:1},{k:'autoclicker',s:1},{k:'reach',s:1},{k:'swinganim',s:1}
        ],
        Movement: [
            {k:'autojump',s:0},{k:'bhop',s:1},{k:'autosprint',s:0},{k:'scaffold',s:1}
        ],
        Visual: [
            {k:'swordspin',s:1},{k:'nametags',s:1},{k:'fullbright',s:0},{k:'reversehead',s:0},{k:'headdown',s:0},
            {k:'tracers',s:1},{k:'custompose',s:1},{k:'viewmodel',s:1},{k:'characteranims',s:1},{k:'arraylist',s:0},{k:'chams',s:0}
        ],
        Misc: [
            {k:'newaccount',s:0}
        ]
    };

    function getDisplayVal(hack, setting) {
        if (setting.display && Array.isArray(setting.display)) {
            const idx = Math.floor(setting.val) - (setting.min || 0);
            return setting.display[idx] || setting.val;
        }
        return setting.val;
    }

    categories.forEach(cn => {
        const cw = document.createElement('div');
        cw.style.cssText = `display:flex;flex-direction:column;align-items:center;`;

        const btn = document.createElement('button');
        btn.textContent = cn;
        btn.style.cssText = `background:transparent;color:white;border:2px solid rgba(255,255,255,0.3);padding:${20*SCALE}px ${30*SCALE}px;font-size:${22*SCALE}px;font-weight:bold;cursor:pointer;border-radius:8px;letter-spacing:1px;transition:all 0.2s;min-width:${130*SCALE}px;backdrop-filter:blur(5px);white-space:nowrap;`;
        bp[cn] = { x:0, y:0 };
        btn.addEventListener('mouseenter', () => { btn.style.background = 'rgba(255,255,255,0.2)'; btn.style.borderColor = 'rgba(255,255,255,0.6)'; });
        btn.addEventListener('mouseleave', () => { btn.style.background = 'transparent'; btn.style.borderColor = 'rgba(255,255,255,0.3)'; });

        const sm = document.createElement('div');
        sm.style.cssText = `margin-top:${8*SCALE}px;display:flex;flex-direction:column;gap:${2*SCALE}px;width:100%;`;

        const hks = ch[cn] || [];
        hks.forEach(hd => {
            const hk = Hacks[hd.k];
            if (!hk) return;

            const hr = document.createElement('div');
            hr.style.cssText = `display:flex;align-items:center;width:100%;gap:${6*SCALE}px;`;

            const hb = document.createElement('button');
            hb.textContent = hk.name;
            hb.style.cssText = `background:transparent;color:white;border:2px solid rgba(255,255,255,0.3);padding:${10*SCALE}px ${15*SCALE}px;font-size:${15*SCALE}px;font-weight:bold;cursor:pointer;border-radius:6px;letter-spacing:1px;transition:all 0.3s;flex:1;backdrop-filter:blur(5px);text-align:left;`;

            let kb = null;
            if (!isMobile) {
                kb = document.createElement('button');
                kb.textContent = hk.keybind || 'None';
                kb.style.cssText = `background:rgba(0,0,0,0.5);color:rgba(255,255,255,0.7);border:1px solid rgba(255,255,255,0.2);padding:${6*SCALE}px ${8*SCALE}px;font-size:${10*SCALE}px;font-weight:bold;cursor:pointer;border-radius:4px;letter-spacing:1px;transition:all 0.2s;backdrop-filter:blur(5px);flex-shrink:0;min-width:${38*SCALE}px;text-align:center;font-family:monospace;margin-left:${38*SCALE}px;`;
            }

            let ab = null, ho = null;
            if (hd.s && hk.settings && hk.settings.length > 0) {
                ab = document.createElement('button');
                ab.textContent = '▶';
                ab.style.cssText = `background:transparent;color:rgba(255,255,255,0.6);border:2px solid rgba(255,255,255,0.3);padding:${10*SCALE}px ${10*SCALE}px;font-size:${13*SCALE}px;font-weight:bold;cursor:pointer;border-radius:6px;transition:all 0.3s;backdrop-filter:blur(5px);flex-shrink:0;min-width:${32*SCALE}px;`;
                ho = document.createElement('div');
                ho.style.cssText = `display:none;flex-direction:column;gap:${6*SCALE}px;background:rgba(0,0,0,0.7);border:1px solid rgba(255,255,255,0.2);border-radius:6px;padding:${8*SCALE}px;margin-top:${4*SCALE}px;width:100%;transition:all 0.3s cubic-bezier(0.68,-0.55,0.265,1.55);transform:perspective(500px) rotateX(-90deg);transform-origin:top center;opacity:0;`;
            }

            let ex = false;
            hb.addEventListener('click', (e) => {
                e.stopPropagation();
                hk.enabled = !hk.enabled;
                if (hk.onChange) hk.onChange.call(hk, hk.enabled);
                updateBtnStyle();
                HUD.update();
                resetRainbowSync();
                saveSettings();
            });

            if (kb) {
                kb.addEventListener('click', (e) => {
                    e.stopPropagation();
                    const oldText = kb.textContent;
                    kb.textContent = '...';
                    kb.style.background = 'rgba(255,100,0,0.5)';
                    const keyHandler = (ev) => {
                        ev.stopPropagation(); ev.preventDefault();
                        let newKey = ev.key.toUpperCase();
                        if (newKey === 'ESCAPE') { kb.textContent = oldText; }
                        else if (newKey === 'BACKSPACE' || newKey === 'DELETE') { hk.keybind = 'None'; kb.textContent = 'None'; }
                        else if (newKey.length === 1 || newKey.startsWith('F')) { hk.keybind = newKey; kb.textContent = newKey; }
                        else { kb.textContent = oldText; }
                        kb.style.background = 'rgba(0,0,0,0.5)';
                        window.removeEventListener('keydown', keyHandler, true);
                        saveSettings();
                    };
                    window.addEventListener('keydown', keyHandler, true);
                });
            }

            if (ab && ho) {
                ab.addEventListener('click', (e) => {
                    e.stopPropagation();
                    ex = !ex;
                    ho.style.display = ex ? 'flex' : 'none';
                    ab.textContent = ex ? '▼' : '▶';
                    if (ex) {
                        setTimeout(() => { ho.style.transform = 'perspective(500px) rotateX(0deg)'; ho.style.opacity = '1'; }, 10);
                    } else {
                        ho.style.transform = 'perspective(500px) rotateX(-90deg)'; ho.style.opacity = '0';
                    }
                });
            }

            function updateBtnStyle() {
                if (hk.enabled) {
                    hb.style.background = 'linear-gradient(90deg,#00BFFF,#FFD700,#FF4444,#00BFFF)';
                    hb.style.backgroundSize = '300% 100%';
                    hb.style.color = 'white';
                    hb.style.borderColor = 'transparent';
                    hb.style.animation = 'wjc-rb 2s linear infinite';
                    hb.style.textShadow = '1px 1px 2px rgba(0,0,0,0.5)';
                } else {
                    hb.style.background = 'transparent';
                    hb.style.color = 'white';
                    hb.style.borderColor = 'rgba(255,255,255,0.3)';
                    hb.style.animation = 'none';
                    hb.style.textShadow = 'none';
                }
            }

            if (ho && hk.settings) {
                hk.settings.forEach((st, idx) => {
                    if (st.type === 'range') {
                        const r = document.createElement('div');
                        r.style.cssText = `display:flex;flex-direction:column;gap:${4*SCALE}px;color:white;font-size:${13*SCALE}px;`;
                        const displayVal = getDisplayVal(hk, st);
                        r.innerHTML = `<div style="display:flex;justify-content:space-between;"><span>${st.name}</span><span id="sv-${hd.k}-${idx}">${displayVal}</span></div><input type="range" min="${st.min}" max="${st.max}" step="${st.step||1}" value="${st.val}" data-mod="${hd.k}" data-idx="${idx}" style="width:100%;accent-color:#00BFFF;">`;
                        const vs = r.querySelector('span:last-child');
                        const slider = r.querySelector('input');
                        slider.addEventListener('input', (e) => {
                            st.val = parseFloat(e.target.value);
                            vs.textContent = getDisplayVal(hk, st);
                        });
                        slider.addEventListener('change', () => saveSettings());
                        ho.appendChild(r);
                    } else if (st.type === 'toggle') {
                        const r = document.createElement('div');
                        r.style.cssText = `display:flex;justify-content:space-between;align-items:center;color:white;font-size:${13*SCALE}px;`;
                        r.innerHTML = `<span>${st.name}</span><div class="wjc-tg" style="width:${40*SCALE}px;height:${20*SCALE}px;background:${st.val?'#00BFFF':'#555'};border-radius:${20*SCALE}px;cursor:pointer;position:relative;"><div style="width:${16*SCALE}px;height:${16*SCALE}px;background:white;border-radius:50%;position:absolute;top:${2*SCALE}px;left:${st.val?22*SCALE:2*SCALE}px;transition:all 0.3s;"></div></div>`;
                        const tg = r.querySelector('.wjc-tg'), cr = tg.querySelector('div');
                        tg.addEventListener('click', () => {
                            st.val = !st.val;
                            tg.style.background = st.val ? '#00BFFF' : '#555';
                            cr.style.left = st.val ? `${22*SCALE}px` : `${2*SCALE}px`;
                            saveSettings();
                        });
                        ho.appendChild(r);
                    }
                });
            }

            hr.appendChild(hb);
            if (kb) hr.appendChild(kb);
            if (ab) hr.appendChild(ab);
            sm.appendChild(hr);
            if (ho) sm.appendChild(ho);
        });

        cw.appendChild(btn);
        cw.appendChild(sm);
        makeDraggable(cw, cn);
        catCont.appendChild(cw);
    });

    function resetRainbowSync() {
        document.querySelectorAll('[style*="wjc-rb"]').forEach(b => {
            b.style.animation = 'none';
            b.offsetHeight;
            b.style.animation = 'wjc-rb 2s linear infinite';
        });
    }

    container.appendChild(catCont);
    document.body.appendChild(container);

    function makeDraggable(el, nm) {
        let d = false, sx, sy;
        el.addEventListener('mousedown', (e) => {
            if (e.target.tagName !== 'INPUT' && !e.target.closest('.wjc-tg')) {
                d = true;
                sx = e.clientX - (bp[nm].x || 0);
                sy = e.clientY - (bp[nm].y || 0);
                el.style.zIndex = '1000000';
                e.preventDefault();
            }
        });
        document.addEventListener('mousemove', (e) => {
            if (d) {
                bp[nm].x = e.clientX - sx;
                bp[nm].y = e.clientY - sy;
                el.style.transform = `translate(${bp[nm].x}px,${bp[nm].y}px)`;
            }
        });
        document.addEventListener('mouseup', () => {
            if (d) { d = false; el.style.zIndex = ''; }
        });
    }

    // ================================================================
    // VISIBILIDAD
    // ================================================================
    let menuVisible = false;
    function openMenu() {
        if (menuVisible) return;
        menuVisible = true;
        overlay.style.display = 'block';
        container.style.display = 'block';
        setTimeout(() => {
            container.style.transform = 'perspective(1000px) rotateX(0deg)';
            container.style.opacity = '1';
        }, 10);
        startRainbow();
    }
    function closeMenu() {
        if (!menuVisible) return;
        menuVisible = false;
        container.style.transform = 'perspective(1000px) rotateX(-90deg)';
        container.style.opacity = '0';
        setTimeout(() => {
            container.style.display = 'none';
            overlay.style.display = 'none';
        }, 400);
        stopRainbow();
    }
    function toggleMenu() {
        menuVisible ? closeMenu() : openMenu();
    }

    document.addEventListener('keydown', (e) => {
        if (e.code === 'ShiftRight') {
            e.preventDefault();
            toggleMenu();
            return;
        }
        const key = e.code.replace('Key', '').toUpperCase();
        for (const [hk, h] of Object.entries(Hacks)) {
            if (h.keybind && h.keybind !== 'None' && h.keybind.toUpperCase() === key) {
                h.enabled = !h.enabled;
                if (h.onChange) h.onChange.call(h, h.enabled);
                HUD.update();
                resetRainbowSync();
                saveSettings();
            }
        }
    });

    overlay.addEventListener('click', () => closeMenu());

    // ================================================================
    // RAINBOW
    // ================================================================
    const rc = ['#00BFFF','#FFD700','#FF4444'];
    let rp = 0, ra;
    function startRainbow() { if (!ra) { rp = 0; animR(); } }
    function stopRainbow() { if (ra) { cancelAnimationFrame(ra); ra = null; } }
    function animR() {
        const t = document.getElementById('wjc-title');
        if (!t) return;
        const seg = rc.length, si = Math.floor(rp), ni = (si+1)%seg, p = rp - si;
        const c1 = hexToRgb(rc[si]), c2 = hexToRgb(rc[ni]);
        t.style.color = `rgb(${Math.round(c1.r+(c2.r-c1.r)*p)},${Math.round(c1.g+(c2.g-c1.g)*p)},${Math.round(c1.b+(c2.b-c1.b)*p)})`;
        rp += 0.02;
        if (rp >= seg) rp = 0;
        ra = requestAnimationFrame(animR);
    }
    function hexToRgb(h) {
        const r = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(h);
        return r ? { r: parseInt(r[1],16), g: parseInt(r[2],16), b: parseInt(r[3],16) } : { r:0,g:0,b:0 };
    }
    const bs = document.createElement('style');
    bs.textContent = `@keyframes wjc-rb{0%{background-position:0% 50%}100%{background-position:300% 50%}}`;
    document.head.appendChild(bs);

    // ================================================================
    // GUARDADO
    // ================================================================
    function saveSettings() {
        const data = {};
        for (const key in Hacks) {
            data[key] = {
                enabled: Hacks[key].enabled,
                keybind: Hacks[key].keybind,
                settings: Hacks[key].settings ? Hacks[key].settings.map(s => ({ name: s.name, val: s.val })) : []
            };
        }
        localStorage.setItem('wolf_juan_settings', JSON.stringify(data));
    }

    function loadSettings() {
        try {
            const saved = localStorage.getItem('wolf_juan_settings');
            if (!saved) return;
            const data = JSON.parse(saved);
            for (const key in data) {
                if (Hacks[key]) {
                    Hacks[key].enabled = data[key].enabled || false;
                    Hacks[key].keybind = data[key].keybind || 'None';
                    if (data[key].settings && Hacks[key].settings) {
                        data[key].settings.forEach(savedS => {
                            const currentS = Hacks[key].settings.find(s => s.name === 'SavedS.name');
                            if (currentS) currentS.val = savedS.val;
                        });
                    }
                    if (Hacks[key].enabled && Hacks[key].onChange) {
                        Hacks[key].onChange.call(Hacks[key], true);
                    }
                }
            }
        } catch(e) {}
    }

    // ================================================================
    // GAME LOOP
    // ================================================================
    HUD.init();
    loadSettings();

    setInterval(() => {
        if (!G.n) { G.init(); return; }
        for (const key in Hacks) {
            const hack = Hacks[key];
            if (hack.enabled && hack.onTick) {
                hack.onTick.call(hack);
            }
        }
        HUD.update();
    }, 20);

    startRainbow();
    console.log('%c🐺 Wolf Juan Client %c¡Funcionando! Clic en el lobo o Shift Derecho.', 'color:#00BFFF;font-size:16px;', 'color:#FFD700;');
})();
