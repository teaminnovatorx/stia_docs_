<!-- STIA Portal — Bento Grid Interface -->

<section class="min-h-screen flex items-center justify-center py-20 px-4">
  <div class="max-w-6xl w-full">
    
    <!-- Hero Header -->
    <div class="text-center mb-16 space-y-4 animate-in fade-in slide-in-from-bottom-8 duration-1000">
      <div class="inline-flex items-center space-x-2 px-3 py-1 rounded-full bg-cyan-500/10 border border-cyan-500/20 text-cyan-400 text-xs font-bold tracking-[0.2em] uppercase">
        <span class="relative flex h-2 w-2">
          <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-cyan-400 opacity-75"></span>
          <span class="relative inline-flex rounded-full h-2 w-2 bg-cyan-500"></span>
        </span>
        <span>Neural Uplink Active</span>
      </div>
      <h1 class="text-5xl md:text-7xl font-extrabold text-white tracking-tighter">
        STIA <span class="text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-500">MATRIX</span>
      </h1>
      <p class="text-slate-400 max-w-2xl mx-auto text-lg md:text-xl font-medium leading-relaxed">
        Technical Intelligence Agency Command Center for UDARA AI. <br class="hidden md:block">
        Offline-first AMR surveillance for sub-Saharan Africa.
      </p>
    </div>

    <!-- Bento Grid -->
    <div class="grid grid-cols-1 md:grid-cols-4 lg:grid-cols-6 gap-4 auto-rows-[180px]">
      
      <!-- Primary Action -->
      <a href="#/01-architecture-overview" class="md:col-span-2 lg:col-span-3 row-span-2 group relative overflow-hidden rounded-3xl border border-white/10 bg-slate-900/50 p-8 flex flex-col justify-end transition-all hover:border-cyan-500/50 hover:shadow-[0_0_30px_rgba(34,211,238,0.1)]">
        <div class="absolute top-0 right-0 p-8 text-cyan-500/20 group-hover:text-cyan-400/40 transition-colors">
          <i data-lucide="zap" class="w-32 h-32 rotate-12 translate-x-10 -translate-y-10"></i>
        </div>
        <div class="relative z-10 space-y-2">
          <div class="text-cyan-400 font-bold tracking-widest text-xs uppercase">Initiate Protocol</div>
          <h2 class="text-3xl font-bold text-white leading-tight">Architecture <br>Overview</h2>
          <p class="text-slate-400 text-sm max-w-[200px]">Begin system exploration and architectural deep-dive.</p>
        </div>
      </a>

      <!-- System Status -->
      <div class="md:col-span-2 lg:col-span-3 row-span-1 rounded-3xl border border-white/10 bg-slate-900/50 p-6 flex items-center space-x-6">
        <div class="flex-shrink-0 w-16 h-16 rounded-2xl bg-cyan-500/10 border border-cyan-500/20 flex items-center justify-center text-cyan-400">
          <i data-lucide="cpu" class="w-8 h-8"></i>
        </div>
        <div>
          <div class="text-[10px] text-slate-500 uppercase tracking-widest font-bold mb-1">Local Edge Processing</div>
          <div class="text-xl font-bold text-slate-200">8GB LPDDR4X</div>
          <div class="text-xs text-cyan-500/80 font-medium flex items-center space-x-1 mt-1">
             <i data-lucide="check-circle" class="w-3 h-3"></i>
             <span>Optimized for RPi 5</span>
          </div>
        </div>
      </div>

      <!-- Network Stats -->
      <div class="md:col-span-1 lg:col-span-1 row-span-1 rounded-3xl border border-white/10 bg-slate-900/50 p-6 flex flex-col justify-between">
        <div class="text-cyan-400"><i data-lucide="globe"></i></div>
        <div>
          <div class="text-2xl font-bold text-white">48</div>
          <div class="text-[10px] text-slate-500 uppercase tracking-widest font-bold">Member States</div>
        </div>
      </div>

      <!-- Tech Stack -->
      <div class="md:col-span-1 lg:col-span-2 row-span-1 rounded-3xl border border-white/10 bg-slate-900/50 p-6 flex flex-col justify-between group cursor-default">
        <div class="flex items-center justify-between">
          <span class="text-cyan-400"><i data-lucide="layers"></i></span>
          <span class="text-[10px] bg-white/5 text-slate-400 px-2 py-1 rounded border border-white/5 font-bold uppercase tracking-widest">Stack</span>
        </div>
        <div class="flex flex-wrap gap-2 mt-4">
          <span class="text-[9px] px-2 py-0.5 rounded-full bg-slate-800 text-slate-300 border border-white/5">Python</span>
          <span class="text-[9px] px-2 py-0.5 rounded-full bg-slate-800 text-slate-300 border border-white/5">TypeScript</span>
          <span class="text-[9px] px-2 py-0.5 rounded-full bg-slate-800 text-slate-300 border border-white/5">Docker</span>
          <span class="text-[9px] px-2 py-0.5 rounded-full bg-slate-800 text-slate-300 border border-white/5">AWS</span>
        </div>
      </div>

      <!-- Live Monitoring -->
      <div class="md:col-span-2 lg:col-span-2 row-span-2 rounded-3xl border border-white/10 bg-slate-900/80 p-6 overflow-hidden relative">
        <div class="absolute inset-0 bg-[radial-gradient(circle_at_50%_120%,rgba(34,211,238,0.1),transparent_70%)]"></div>
        <div class="relative z-10 flex flex-col h-full">
           <div class="flex items-center justify-between mb-4">
              <span class="text-xs font-bold text-slate-400 tracking-widest uppercase italic">Real-time Stream</span>
              <span class="flex h-2 w-2 rounded-full bg-red-500 animate-pulse"></span>
           </div>
           <div class="space-y-3 font-mono text-[10px] opacity-40 leading-tight">
              <div>> SYSTEM_BOOT_SEQUENCE: SUCCESS</div>
              <div>> NEURAL_CORE_LOADED: 100%</div>
              <div>> EDGE_NODE_ID: RPI-AF-820</div>
              <div>> SYNC_DAEMON_ACTIVE: TRUE</div>
              <div>> SCANNING_AMR_SIGNATURES...</div>
              <div class="text-cyan-400">> ENCRYPTED_DATA_BURST: [OK]</div>
           </div>
           <div class="mt-auto pt-4 border-t border-white/5">
              <div class="flex items-center space-x-2 text-cyan-400/60 mb-1">
                 <i data-lucide="shield-check" class="w-4 h-4"></i>
                 <span class="text-[10px] font-bold uppercase tracking-widest">Security Protocol</span>
              </div>
              <div class="text-white font-bold">mTLS + JWT Active</div>
           </div>
        </div>
      </div>

      <!-- Secondary Links -->
      <a href="https://github.com/teaminnovatorx/stia_docs_" target="_blank" class="md:col-span-1 lg:col-span-1 row-span-1 rounded-3xl border border-white/10 bg-slate-900/50 p-6 flex flex-col justify-center items-center group transition-all hover:bg-white/5">
        <i data-lucide="github" class="w-8 h-8 text-slate-500 group-hover:text-white transition-colors mb-2"></i>
        <span class="text-[10px] text-slate-500 group-hover:text-slate-200 uppercase tracking-widest font-bold transition-colors">Repository</span>
      </a>

      <!-- Tech Deep Dive -->
      <a href="#/03-tech-stack" class="md:col-span-1 lg:col-span-1 row-span-1 rounded-3xl border border-white/10 bg-slate-900/50 p-6 flex flex-col justify-center items-center group transition-all hover:bg-white/5">
        <i data-lucide="terminal" class="w-8 h-8 text-slate-500 group-hover:text-cyan-400 transition-colors mb-2"></i>
        <span class="text-[10px] text-slate-500 group-hover:text-slate-200 uppercase tracking-widest font-bold transition-colors">Tech Stack</span>
      </a>

    </div>

    <!-- System Footer -->
    <div class="mt-16 text-center text-slate-600 font-mono text-[10px] tracking-[0.3em] uppercase opacity-40">
      Security Clearance: Level 4 • STIA Command Matrix • 2026 Edition
    </div>

  </div>
</section>
