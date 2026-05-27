import { useState, useEffect, useRef, useCallback } from "react";

// ─── Utilities ────────────────────────────────────────────────────────────────
const formatTime = (date) =>
  date.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });

const generateId = () => Math.random().toString(36).slice(2, 9);

// ─── Static data ─────────────────────────────────────────────────────────────
const CHAT_HISTORY = [
  { id: "h1", title: "Explain quantum entanglement", date: "Today" },
  { id: "h2", title: "React hooks deep dive", date: "Today" },
  { id: "h3", title: "Best pasta recipes", date: "Yesterday" },
  { id: "h4", title: "Stoicism & modern life", date: "Yesterday" },
  { id: "h5", title: "Refactor my auth module", date: "Mon" },
  { id: "h6", title: "Plan a trip to Kyoto", date: "Mon" },
];

const PLACEHOLDER_REPLIES = [
  "That's a great question. Let me think through this carefully — there are several angles worth exploring here, and I want to make sure I give you a thorough, nuanced answer.",
  "Interesting point. Here's how I'd frame it: at the core, the concept you're describing touches on a tension between two competing principles that shows up across many domains.",
  "Sure! The short answer is yes, but the long answer involves a few important caveats that are easy to overlook. Let me walk you through the key points.",
  "Great prompt. I'll break this into three parts so it's easy to follow: first the what, then the why, then some practical takeaways you can actually use.",
  "This is one of those topics where intuition and reality diverge in surprising ways. Let me start by challenging the most common assumption people bring to it.",
];

// ─── Icons (inline SVG, no external deps) ────────────────────────────────────
const IconSend = () => (
  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M22 2L11 13" /><path d="M22 2L15 22l-4-9-9-4 20-7z" />
  </svg>
);
const IconPlus = () => (
  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M12 5v14M5 12h14" />
  </svg>
);
const IconCopy = () => (
  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <rect x="9" y="9" width="13" height="13" rx="2" /><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
  </svg>
);
const IconCheck = () => (
  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M20 6L9 17l-5-5" />
  </svg>
);
const IconMenu = () => (
  <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M4 6h16M4 12h16M4 18h16" />
  </svg>
);
const IconMic = () => (
  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <rect x="9" y="2" width="6" height="12" rx="3" /><path d="M5 10a7 7 0 0 0 14 0" /><path d="M12 19v3" /><path d="M9 22h6" />
  </svg>
);
const IconAttach = () => (
  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M21.44 11.05l-9.19 9.19a6 6 0 0 1-8.49-8.49l9.19-9.19a4 4 0 0 1 5.66 5.66L9.64 16.2a2 2 0 0 1-2.83-2.83l8.49-8.48" />
  </svg>
);
const IconBot = () => (
  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <rect x="3" y="11" width="18" height="10" rx="2" /><circle cx="12" cy="5" r="2" /><path d="M12 7v4" /><path d="M8 15h.01M16 15h.01" />
  </svg>
);
const IconChevron = () => (
  <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M6 9l6 6 6-6" />
  </svg>
);
const IconTrash = () => (
  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
    <path d="M3 6h18M8 6V4h8v2M19 6l-1 14H6L5 6" />
  </svg>
);

// ─── Typing dots ──────────────────────────────────────────────────────────────
const TypingDots = () => (
  <div style={{ display: "flex", gap: 5, alignItems: "center", padding: "4px 0" }}>
    {[0, 1, 2].map((i) => (
      <span key={i} style={{
        width: 7, height: 7, borderRadius: "50%",
        background: "var(--accent)",
        animation: `typingBounce 1.2s ease-in-out ${i * 0.2}s infinite`,
        display: "block",
      }} />
    ))}
  </div>
);

// ─── Message bubble ───────────────────────────────────────────────────────────
const MessageBubble = ({ msg }) => {
  const [copied, setCopied] = useState(false);
  const isUser = msg.role === "user";

  const handleCopy = () => {
    navigator.clipboard.writeText(msg.content).catch(() => {});
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <div style={{
      display: "flex",
      flexDirection: isUser ? "row-reverse" : "row",
      gap: 10,
      alignItems: "flex-start",
      animation: "fadeSlideIn 0.25s ease forwards",
      opacity: 0,
    }}>
      {/* Avatar */}
      {!isUser && (
        <div style={{
          width: 32, height: 32, borderRadius: "50%", flexShrink: 0,
          background: "linear-gradient(135deg, var(--accent), var(--accent2))",
          display: "flex", alignItems: "center", justifyContent: "center",
          color: "#fff", marginTop: 2,
        }}>
          <IconBot />
        </div>
      )}

      <div style={{ maxWidth: "72%", display: "flex", flexDirection: "column", gap: 4, alignItems: isUser ? "flex-end" : "flex-start" }}>
        <div style={{
          padding: "11px 15px",
          borderRadius: isUser ? "18px 18px 4px 18px" : "18px 18px 18px 4px",
          background: isUser
            ? "linear-gradient(135deg, var(--accent), var(--accent2))"
            : "var(--bubble-ai)",
          color: isUser ? "#fff" : "var(--text-primary)",
          fontSize: 14.5,
          lineHeight: 1.65,
          border: isUser ? "none" : "1px solid var(--border)",
          backdropFilter: "blur(8px)",
          boxShadow: isUser
            ? "0 4px 20px rgba(99,102,241,0.25)"
            : "0 2px 12px rgba(0,0,0,0.15)",
        }}>
          {msg.content}
        </div>

        {/* Timestamp + copy */}
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <span style={{ fontSize: 11, color: "var(--text-muted)", fontFamily: "var(--font-mono, monospace)" }}>
            {formatTime(msg.timestamp)}
          </span>
          {!isUser && (
            <button onClick={handleCopy} title="Copy" style={{
              background: "none", border: "none", cursor: "pointer",
              color: copied ? "var(--success)" : "var(--text-muted)",
              display: "flex", alignItems: "center", gap: 4,
              fontSize: 11, padding: "2px 4px", borderRadius: 4,
              transition: "color 0.2s",
            }}>
              {copied ? <IconCheck /> : <IconCopy />}
              {copied ? "Copied" : "Copy"}
            </button>
          )}
        </div>
      </div>
    </div>
  );
};

// ─── Sidebar ──────────────────────────────────────────────────────────────────
const Sidebar = ({ open, onClose, activeChat, onSelectChat, onNewChat }) => {
  const grouped = [
    { label: "Today", items: CHAT_HISTORY.filter(c => c.date === "Today") },
    { label: "Yesterday", items: CHAT_HISTORY.filter(c => c.date === "Yesterday") },
    { label: "Earlier", items: CHAT_HISTORY.filter(c => c.date === "Mon") },
  ];

  return (
    <>
      {/* Overlay (mobile) */}
      {open && (
        <div onClick={onClose} style={{
          position: "fixed", inset: 0, background: "rgba(0,0,0,0.5)",
          zIndex: 40, display: "none",
        }} className="sidebar-overlay" />
      )}

      <aside style={{
        width: 260, flexShrink: 0,
        background: "var(--sidebar-bg)",
        borderRight: "1px solid var(--border)",
        display: "flex", flexDirection: "column",
        height: "100%",
        transition: "transform 0.3s cubic-bezier(0.4,0,0.2,1)",
        position: "relative", zIndex: 50,
      }}>
        {/* Logo */}
        <div style={{ padding: "18px 16px 12px", borderBottom: "1px solid var(--border)" }}>
          <div style={{ display: "flex", alignItems: "center", gap: 9, marginBottom: 14 }}>
            <div style={{
              width: 30, height: 30, borderRadius: 8,
              background: "linear-gradient(135deg, var(--accent), var(--accent2))",
              display: "flex", alignItems: "center", justifyContent: "center",
            }}>
              <IconBot />
            </div>
            <span style={{ fontSize: 15, fontWeight: 600, color: "var(--text-primary)", letterSpacing: "-0.3px" }}>
              NeuralChat
            </span>
          </div>
          <button onClick={onNewChat} style={{
            width: "100%", padding: "9px 12px",
            background: "linear-gradient(135deg, var(--accent), var(--accent2))",
            border: "none", borderRadius: 10, cursor: "pointer",
            color: "#fff", fontSize: 13.5, fontWeight: 500,
            display: "flex", alignItems: "center", gap: 7,
            justifyContent: "center",
            boxShadow: "0 4px 16px rgba(99,102,241,0.3)",
            transition: "opacity 0.15s, transform 0.15s",
          }}
            onMouseEnter={e => { e.currentTarget.style.opacity = "0.88"; e.currentTarget.style.transform = "translateY(-1px)"; }}
            onMouseLeave={e => { e.currentTarget.style.opacity = "1"; e.currentTarget.style.transform = "translateY(0)"; }}
          >
            <IconPlus /> New chat
          </button>
        </div>

        {/* History */}
        <div style={{ flex: 1, overflowY: "auto", padding: "10px 8px" }}>
          {grouped.map(group => (
            <div key={group.label} style={{ marginBottom: 16 }}>
              <p style={{ fontSize: 10.5, fontWeight: 600, color: "var(--text-muted)", letterSpacing: "0.06em", textTransform: "uppercase", padding: "4px 8px 6px" }}>
                {group.label}
              </p>
              {group.items.map(chat => (
                <button key={chat.id} onClick={() => onSelectChat(chat.id)} style={{
                  width: "100%", textAlign: "left", padding: "8px 10px",
                  borderRadius: 8, border: "none", cursor: "pointer",
                  background: activeChat === chat.id ? "var(--item-active)" : "transparent",
                  color: activeChat === chat.id ? "var(--text-primary)" : "var(--text-secondary)",
                  fontSize: 13.5, display: "flex", alignItems: "center",
                  justifyContent: "space-between", gap: 6,
                  transition: "background 0.15s, color 0.15s",
                  borderLeft: activeChat === chat.id ? "2px solid var(--accent)" : "2px solid transparent",
                }}
                  onMouseEnter={e => { if (activeChat !== chat.id) e.currentTarget.style.background = "var(--item-hover)"; }}
                  onMouseLeave={e => { if (activeChat !== chat.id) e.currentTarget.style.background = "transparent"; }}
                >
                  <span style={{ overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap", flex: 1 }}>
                    {chat.title}
                  </span>
                  {activeChat === chat.id && (
                    <span style={{ opacity: 0.4, flexShrink: 0 }}><IconTrash /></span>
                  )}
                </button>
              ))}
            </div>
          ))}
        </div>

        {/* User */}
        <div style={{ padding: "12px 14px", borderTop: "1px solid var(--border)" }}>
          <div style={{ display: "flex", alignItems: "center", gap: 10, cursor: "pointer", padding: "6px 4px", borderRadius: 8, transition: "background 0.15s" }}
            onMouseEnter={e => e.currentTarget.style.background = "var(--item-hover)"}
            onMouseLeave={e => e.currentTarget.style.background = "transparent"}
          >
            <div style={{
              width: 32, height: 32, borderRadius: "50%",
              background: "linear-gradient(135deg, #f97316, #ec4899)",
              display: "flex", alignItems: "center", justifyContent: "center",
              fontSize: 13, fontWeight: 700, color: "#fff", flexShrink: 0,
            }}>A</div>
            <div style={{ flex: 1, overflow: "hidden" }}>
              <p style={{ fontSize: 13.5, fontWeight: 500, color: "var(--text-primary)", margin: 0, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>Alex Johnson</p>
              <p style={{ fontSize: 11.5, color: "var(--text-muted)", margin: 0 }}>Free plan</p>
            </div>
            <IconChevron />
          </div>
        </div>
      </aside>
    </>
  );
};

// ─── Main app ─────────────────────────────────────────────────────────────────
export default function App() {
  const [messages, setMessages] = useState([
    {
      id: generateId(), role: "assistant",
      content: "Hello! I'm your AI assistant. I can help you write, analyze, code, brainstorm, and much more. What's on your mind?",
      timestamp: new Date(),
    }
  ]);
  const [input, setInput] = useState("");
  const [isTyping, setIsTyping] = useState(false);
  const [sidebarOpen, setSidebarOpen] = useState(true);
  const [activeChat, setActiveChat] = useState("h1");
  const bottomRef = useRef(null);
  const inputRef = useRef(null);
  const replyIndex = useRef(0);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, isTyping]);

  const sendMessage = useCallback(() => {
    const text = input.trim();
    if (!text || isTyping) return;
    setInput("");

    const userMsg = { id: generateId(), role: "user", content: text, timestamp: new Date() };
    setMessages(prev => [...prev, userMsg]);
    setIsTyping(true);

    // Simulate AI response with realistic delay
    const delay = 1000 + Math.random() * 1200;
    setTimeout(() => {
      const reply = PLACEHOLDER_REPLIES[replyIndex.current % PLACEHOLDER_REPLIES.length];
      replyIndex.current++;
      setIsTyping(false);
      setMessages(prev => [...prev, { id: generateId(), role: "assistant", content: reply, timestamp: new Date() }]);
    }, delay);
  }, [input, isTyping]);

  const handleKeyDown = (e) => {
    if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); sendMessage(); }
  };

  const handleNewChat = () => {
    setMessages([{ id: generateId(), role: "assistant", content: "New conversation started. What can I help you with?", timestamp: new Date() }]);
    setInput("");
    setIsTyping(false);
    setActiveChat(null);
  };

  return (
    <>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@300;400;500;600&family=JetBrains+Mono:wght@400&display=swap');

        *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

        :root {
          --bg: #0c0c0e;
          --sidebar-bg: #111114;
          --surface: #16161a;
          --bubble-ai: rgba(255,255,255,0.04);
          --border: rgba(255,255,255,0.08);
          --text-primary: #f0f0f4;
          --text-secondary: #9898a8;
          --text-muted: #5c5c70;
          --accent: #6366f1;
          --accent2: #8b5cf6;
          --success: #34d399;
          --item-hover: rgba(255,255,255,0.04);
          --item-active: rgba(99,102,241,0.12);
          --input-bg: rgba(255,255,255,0.05);
          --font-mono: 'JetBrains Mono', monospace;
        }

        body { background: var(--bg); color: var(--text-primary); font-family: 'DM Sans', system-ui, sans-serif; }

        #root { display: flex; height: 100vh; overflow: hidden; }

        ::-webkit-scrollbar { width: 4px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.1); border-radius: 4px; }

        textarea:focus { outline: none; }
        textarea { resize: none; }

        @keyframes fadeSlideIn {
          from { opacity: 0; transform: translateY(8px); }
          to   { opacity: 1; transform: translateY(0); }
        }

        @keyframes typingBounce {
          0%, 60%, 100% { transform: translateY(0); opacity: 0.4; }
          30%            { transform: translateY(-5px); opacity: 1; }
        }

        @keyframes pulse {
          0%, 100% { opacity: 1; }
          50% { opacity: 0.5; }
        }

        .send-btn:hover { opacity: 0.85; transform: scale(1.05); }
        .send-btn:active { transform: scale(0.97); }
        .send-btn { transition: opacity 0.15s, transform 0.15s; }

        @media (max-width: 680px) {
          .sidebar { display: none; }
          .sidebar-open { display: flex !important; position: fixed !important; top: 0; left: 0; height: 100% !important; z-index: 60; }
        }
      `}</style>

      {/* Sidebar */}
      <div className={`sidebar ${sidebarOpen ? "sidebar-open" : ""}`} style={{ display: sidebarOpen ? "flex" : "none", height: "100%" }}>
        <Sidebar
          open={sidebarOpen}
          onClose={() => setSidebarOpen(false)}
          activeChat={activeChat}
          onSelectChat={setActiveChat}
          onNewChat={handleNewChat}
        />
      </div>

      {/* Main */}
      <main style={{ flex: 1, display: "flex", flexDirection: "column", height: "100%", overflow: "hidden", background: "var(--bg)" }}>

        {/* Header */}
        <header style={{
          padding: "14px 20px", borderBottom: "1px solid var(--border)",
          display: "flex", alignItems: "center", gap: 14,
          backdropFilter: "blur(12px)",
          background: "rgba(12,12,14,0.85)",
          flexShrink: 0,
        }}>
          <button onClick={() => setSidebarOpen(v => !v)} style={{
            background: "none", border: "1px solid var(--border)", borderRadius: 8,
            padding: "6px 8px", cursor: "pointer", color: "var(--text-secondary)",
            display: "flex", transition: "background 0.15s, color 0.15s",
          }}
            onMouseEnter={e => { e.currentTarget.style.background = "var(--item-hover)"; e.currentTarget.style.color = "var(--text-primary)"; }}
            onMouseLeave={e => { e.currentTarget.style.background = "none"; e.currentTarget.style.color = "var(--text-secondary)"; }}
          >
            <IconMenu />
          </button>

          <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
            <div style={{
              width: 8, height: 8, borderRadius: "50%",
              background: "var(--success)",
              boxShadow: "0 0 8px rgba(52,211,153,0.6)",
              animation: "pulse 2.5s ease infinite",
            }} />
            <span style={{ fontSize: 14, fontWeight: 500, color: "var(--text-primary)" }}>AI Assistant</span>
            <span style={{
              fontSize: 11, padding: "2px 8px", borderRadius: 20,
              background: "rgba(99,102,241,0.15)", color: "var(--accent)",
              border: "1px solid rgba(99,102,241,0.25)", fontWeight: 500,
            }}>GPT-4o</span>
          </div>

          <div style={{ marginLeft: "auto", fontSize: 12.5, color: "var(--text-muted)" }}>
            {messages.length - 1} message{messages.length !== 2 ? "s" : ""}
          </div>
        </header>

        {/* Messages */}
        <div style={{ flex: 1, overflowY: "auto", padding: "28px 20px", display: "flex", flexDirection: "column", gap: 20 }}>
          <div style={{ maxWidth: 740, width: "100%", margin: "0 auto", display: "flex", flexDirection: "column", gap: 20 }}>
            {messages.map(msg => <MessageBubble key={msg.id} msg={msg} />)}

            {/* Typing indicator */}
            {isTyping && (
              <div style={{ display: "flex", gap: 10, alignItems: "flex-start", animation: "fadeSlideIn 0.2s ease forwards" }}>
                <div style={{
                  width: 32, height: 32, borderRadius: "50%", flexShrink: 0,
                  background: "linear-gradient(135deg, var(--accent), var(--accent2))",
                  display: "flex", alignItems: "center", justifyContent: "center", color: "#fff",
                }}>
                  <IconBot />
                </div>
                <div style={{
                  padding: "11px 16px", borderRadius: "18px 18px 18px 4px",
                  background: "var(--bubble-ai)", border: "1px solid var(--border)",
                }}>
                  <TypingDots />
                </div>
              </div>
            )}
            <div ref={bottomRef} />
          </div>
        </div>

        {/* Input */}
        <div style={{ padding: "14px 20px 18px", borderTop: "1px solid var(--border)", flexShrink: 0, background: "rgba(12,12,14,0.9)", backdropFilter: "blur(16px)" }}>
          <div style={{ maxWidth: 740, margin: "0 auto" }}>
            <div style={{
              display: "flex", alignItems: "flex-end", gap: 10,
              background: "var(--input-bg)",
              border: "1px solid var(--border)",
              borderRadius: 14, padding: "10px 12px",
              transition: "border-color 0.2s, box-shadow 0.2s",
            }}
              onFocusCapture={e => { e.currentTarget.style.borderColor = "rgba(99,102,241,0.5)"; e.currentTarget.style.boxShadow = "0 0 0 3px rgba(99,102,241,0.08)"; }}
              onBlurCapture={e => { e.currentTarget.style.borderColor = "var(--border)"; e.currentTarget.style.boxShadow = "none"; }}
            >
              {/* Attach */}
              <button style={{
                background: "none", border: "none", cursor: "pointer",
                color: "var(--text-muted)", padding: "4px", borderRadius: 6,
                display: "flex", transition: "color 0.15s",
              }}
                onMouseEnter={e => e.currentTarget.style.color = "var(--text-secondary)"}
                onMouseLeave={e => e.currentTarget.style.color = "var(--text-muted)"}
              ><IconAttach /></button>

              <textarea
                ref={inputRef}
                value={input}
                onChange={e => {
                  setInput(e.target.value);
                  e.target.style.height = "auto";
                  e.target.style.height = Math.min(e.target.scrollHeight, 160) + "px";
                }}
                onKeyDown={handleKeyDown}
                placeholder="Message AI Assistant…"
                rows={1}
                style={{
                  flex: 1, background: "none", border: "none",
                  color: "var(--text-primary)", fontSize: 14.5,
                  lineHeight: 1.6, fontFamily: "'DM Sans', system-ui, sans-serif",
                  maxHeight: 160, overflowY: "auto",
                  caretColor: "var(--accent)",
                }}
              />

              {/* Mic */}
              <button style={{
                background: "none", border: "none", cursor: "pointer",
                color: "var(--text-muted)", padding: "4px", borderRadius: 6,
                display: "flex", transition: "color 0.15s",
              }}
                onMouseEnter={e => e.currentTarget.style.color = "var(--text-secondary)"}
                onMouseLeave={e => e.currentTarget.style.color = "var(--text-muted)"}
              ><IconMic /></button>

              {/* Send */}
              <button className="send-btn" onClick={sendMessage} disabled={!input.trim() || isTyping} style={{
                width: 36, height: 36, borderRadius: 10, border: "none",
                background: input.trim() && !isTyping
                  ? "linear-gradient(135deg, var(--accent), var(--accent2))"
                  : "rgba(255,255,255,0.06)",
                color: input.trim() && !isTyping ? "#fff" : "var(--text-muted)",
                cursor: input.trim() && !isTyping ? "pointer" : "default",
                display: "flex", alignItems: "center", justifyContent: "center",
                boxShadow: input.trim() && !isTyping ? "0 4px 14px rgba(99,102,241,0.35)" : "none",
                flexShrink: 0,
              }}>
                <IconSend />
              </button>
            </div>
            <p style={{ textAlign: "center", fontSize: 11, color: "var(--text-muted)", marginTop: 8 }}>
              Press <kbd style={{ padding: "1px 5px", borderRadius: 4, border: "1px solid var(--border)", fontFamily: "var(--font-mono)", fontSize: 10 }}>Enter</kbd> to send &nbsp;·&nbsp;
              <kbd style={{ padding: "1px 5px", borderRadius: 4, border: "1px solid var(--border)", fontFamily: "var(--font-mono)", fontSize: 10 }}>Shift+Enter</kbd> for new line
            </p>
          </div>
        </div>
      </main>
    </>
  );
}
