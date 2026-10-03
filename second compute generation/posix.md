__Portable Operating System Interface__  -  IEEE 1003.1 standard  -  defines the language interface between application programs (along with command line shells and utility interfaces) and the UNIX operating system.
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 950 620" width="100%" height="100%" style="background:#0b0f19; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;">
  <defs>
    <linearGradient id="blueGrad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" stop-color="#3b82f6" />
      <stop offset="100%" stop-color="#1d4ed8" />
    </linearGradient>
    <marker id="arrow" viewBox="0 0 10 10" refX="6" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 1 L 10 5 L 0 9 z" fill="#64748b" />
    </marker>
    <marker id="arrowBlue" viewBox="0 0 10 10" refX="6" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 1 L 10 5 L 0 9 z" fill="#60a5fa" />
    </marker>
  </defs>

  <!-- Title -->
  <text x="475" y="40" text-anchor="middle" fill="#f8fafc" font-size="20" font-weight="bold">MÔ HÌNH POSIX: VIẾT MỘT LẦN, BIÊN DỊCH MỌI NƠI</text>

  <!-- Top Source Box -->
  <rect x="275" y="70" width="400" height="130" rx="10" fill="#1e293b" stroke="#3b82f6" stroke-width="2"/>
  <text x="475" y="95" text-anchor="middle" fill="#93c5fd" font-size="14" font-weight="bold">POSIX C Source Code (Mã nguồn)</text>
  <rect x="295" y="108" width="360" height="75" rx="6" fill="#0f172a" stroke="#334155"/>
  <text x="310" y="128" fill="#38bdf8" font-size="12" font-family="monospace">#include &lt;unistd.h&gt;</text>
  <text x="310" y="145" fill="#38bdf8" font-size="12" font-family="monospace">int main() {</text>
  <text x="325" y="162" fill="#38bdf8" font-size="12" font-family="monospace">write(1, "Hello POSIX\n", 12); return 0;</text>
  <text x="310" y="176" fill="#38bdf8" font-size="12" font-family="monospace">}</text>

  <!-- Central Splitter Lines -->
  <line x1="475" y1="200" x2="475" y2="240" stroke="#60a5fa" stroke-width="2"/>
  <line x1="160" y1="240" x2="790" y2="240" stroke="#60a5fa" stroke-width="2"/>
  
  <line x1="160" y1="240" x2="160" y2="265" stroke="#60a5fa" stroke-width="2" marker-end="url(#arrowBlue)"/>
  <line x1="475" y1="240" x2="475" y2="265" stroke="#60a5fa" stroke-width="2" marker-end="url(#arrowBlue)"/>
  <line x1="790" y1="240" x2="790" y2="265" stroke="#60a5fa" stroke-width="2" marker-end="url(#arrowBlue)"/>

  <rect x="345" y="228" width="260" height="24" rx="12" fill="#1d4ed8"/>
  <text x="475" y="244" text-anchor="middle" fill="#ffffff" font-size="11" font-weight="600">Write Once, Compile Anywhere</text>

  <!-- Linux Column -->
  <g transform="translate(60, 275)">
    <rect width="200" height="70" rx="8" fill="#1e293b" stroke="#22c55e" stroke-width="1.5"/>
    <text x="100" y="28" text-anchor="middle" fill="#86efac" font-size="13" font-weight="bold">Biên dịch (Linux)</text>
    <text x="100" y="48" text-anchor="middle" fill="#94a3b8" font-size="11">GCC / glibc</text>
    
    <line x1="100" y1="70" x2="100" y2="95" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="100" width="200" height="60" rx="8" fill="#14532d" stroke="#22c55e"/>
    <text x="100" y="126" text-anchor="middle" fill="#f0fdf4" font-size="12" font-weight="bold">Binary ELF</text>
    <text x="100" y="144" text-anchor="middle" fill="#bbf7d0" font-size="10">(Chỉ chạy trên Linux)</text>
    
    <line x1="100" y1="160" x2="100" y2="185" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="190" width="200" height="60" rx="8" fill="#0f172a" stroke="#22c55e"/>
    <text x="100" y="215" text-anchor="middle" fill="#86efac" font-size="13" font-weight="bold">Linux Kernel</text>
    <text x="100" y="235" text-anchor="middle" fill="#cbd5e1" font-size="11">Syscall: sys_write</text>
  </g>

  <!-- macOS Column -->
  <g transform="translate(375, 275)">
    <rect width="200" height="70" rx="8" fill="#1e293b" stroke="#0ea5e9" stroke-width="1.5"/>
    <text x="100" y="28" text-anchor="middle" fill="#7dd3fc" font-size="13" font-weight="bold">Biên dịch (macOS)</text>
    <text x="100" y="48" text-anchor="middle" fill="#94a3b8" font-size="11">Apple Clang / libSystem</text>
    
    <line x1="100" y1="70" x2="100" y2="95" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="100" width="200" height="60" rx="8" fill="#0c4a6e" stroke="#0ea5e9"/>
    <text x="100" y="126" text-anchor="middle" fill="#f0f9ff" font-size="12" font-weight="bold">Binary Mach-O</text>
    <text x="100" y="144" text-anchor="middle" fill="#bae6fd" font-size="10">(Chỉ chạy trên macOS)</text>
    
    <line x1="100" y1="160" x2="100" y2="185" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="190" width="200" height="60" rx="8" fill="#0f172a" stroke="#0ea5e9"/>
    <text x="100" y="215" text-anchor="middle" fill="#7dd3fc" font-size="13" font-weight="bold">macOS (XNU)</text>
    <text x="100" y="235" text-anchor="middle" fill="#cbd5e1" font-size="11">Syscall: sys_write</text>
  </g>

  <!-- BSD Column -->
  <g transform="translate(690, 275)">
    <rect width="200" height="70" rx="8" fill="#1e293b" stroke="#f59e0b" stroke-width="1.5"/>
    <text x="100" y="28" text-anchor="middle" fill="#fde68a" font-size="13" font-weight="bold">Biên dịch (FreeBSD)</text>
    <text x="100" y="48" text-anchor="middle" fill="#94a3b8" font-size="11">Clang / BSD libc</text>
    
    <line x1="100" y1="70" x2="100" y2="95" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="100" width="200" height="60" rx="8" fill="#78350f" stroke="#f59e0b"/>
    <text x="100" y="126" text-anchor="middle" fill="#fffbeb" font-size="12" font-weight="bold">Binary ELF</text>
    <text x="100" y="144" text-anchor="middle" fill="#fef3c7" font-size="10">(Chỉ chạy trên BSD)</text>
    
    <line x1="100" y1="160" x2="100" y2="185" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    
    <rect y="190" width="200" height="60" rx="8" fill="#0f172a" stroke="#f59e0b"/>
    <text x="100" y="215" text-anchor="middle" fill="#fde68a" font-size="13" font-weight="bold">FreeBSD Kernel</text>
    <text x="100" y="235" text-anchor="middle" fill="#cbd5e1" font-size="11">Syscall: sys_write</text>
  </g>
</svg>

![Pasted image 20261001095013](img/Pasted%20image%2020261001095013.png)

