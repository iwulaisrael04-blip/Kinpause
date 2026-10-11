// Adaptateur entre Léo et les données de KinPause.
// Réutilise exactement les mêmes fonctions que /api/explore/feed et /api/rooms/[id]/slots,
// donc Léo voit les mêmes chambres, prix et créneaux que le client sur le site.

import { getAllActiveRoomsForExplore } from "@/lib/queries";
import { getUpcomingRoomSlotsForDate } from "@/lib/roomPageQueries";
import { normalizeText } from "@/lib/exploreQueries";
import { todayInKinshasa, isValidDateStr } from "@/lib/kinshasaTime";

export type SearchArgs = {
  query?: string; // quartier, commune, nom d'hôtel ou de chambre
  max_price_usd?: number;
  amenities?: string[];
  sort?: "rating" | "price_asc" | "price_desc";
};

const MAX_RESULTS = 6;

export async function searchRooms(args: SearchArgs, guestId?: string) {
  const catalog = await getAllActiveRoomsForExplore(guestId);

  const q = normalizeText((args.query ?? "").slice(0, 80));
  const wanted = (args.amenities ?? []).map(normalizeText).filter(Boolean);

  const list = catalog.filter((r) => {
    if (args.max_price_usd !== undefined && !(r.cheapestRate <= args.max_price_usd)) return false;
    if (q) {
      const haystack = normalizeText(`${r.name} ${r.hotelName} ${r.neighborhood} ${r.commune ?? ""}`);
      if (!haystack.includes(q)) return false;
    }
    if (wanted.length > 0 && !wanted.every((w) => r.facilityKeys.some((k) => normalizeText(k).includes(w)))) {
      return false;
    }
    return true;
  });

  if (args.sort === "price_asc") list.sort((a, b) => a.cheapestRate - b.cheapestRate);
  else if (args.sort === "price_desc") list.sort((a, b) => b.cheapestRate - a.cheapestRate);
  else list.sort((a, b) => b.rating - a.rating || b.reviewCount - a.reviewCount || a.name.localeCompare(b.name));

  return {
    total: list.length,
    rooms: list.slice(0, MAX_RESULTS).map((r) => ({
      id: r.id,
      name: r.name,
      hotelName: r.hotelName,
      neighborhood: r.neighborhood,
      commune: r.commune ?? null,
      fromPriceUsd: r.cheapestRate, // prix du créneau le moins cher, pas une disponibilité
      rating: r.rating,
      reviewCount: r.reviewCount,
      facilities: r.facilityKeys,
    })),
  };
}

export async function getRoomSlots(roomId: string, date?: string) {
  const dateStr = date || todayInKinshasa();
  if (!isValidDateStr(dateStr)) throw new Error("Date invalide : format attendu YYYY-MM-DD.");
  const slots = await getUpcomingRoomSlotsForDate(roomId, dateStr);
  return { date: dateStr, slots };
}
import { NextResponse } from "next/server";
import { LEO_TOOLS, SERVER_TOOL_NAMES, buildSystemPrompt } from "@/lib/leo/tools";
import { getRoomSlots, searchRooms } from "@/lib/leo/data";
import { getSession } from "@/lib/auth";

// Conversation sans état : le navigateur renvoie l'historique complet à chaque appel.
// On exécute ici les outils serveur ; dès que Claude demande un outil client
// (navigation, wallet, réservation), on rend la main au navigateur.

type Block = { type: string; [k: string]: unknown };
type Msg = { role: "user" | "assistant"; content: string | Block[] };
type ToolUse = { type: "tool_use"; id: string; name: string; input: Record<string, unknown> };

const MODEL = process.env.LEO_MODEL || "claude-sonnet-5-5";
const MAX_LOOPS = 6;
const MAX_MESSAGES = 40;

function kinshasaNow() {
  const now = new Date();
  const fmt = (o: Intl.DateTimeFormatOptions) =>
    new Intl.DateTimeFormat("fr-FR", { timeZone: "Africa/Kinshasa", ...o }).format(now);
  const iso = new Intl.DateTimeFormat("en-CA", { timeZone: "Africa/Kinshasa" }).format(now); // YYYY-MM-DD
  return {
    today: `${fmt({ weekday: "long", day: "numeric", month: "long", year: "numeric" })} (${iso})`,
    time: fmt({ hour: "2-digit", minute: "2-digit" }),
  };
}

async function callClaude(messages: Msg[], loggedIn: boolean) {
  const { today, time } = kinshasaNow();
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY ?? "",
      "anthropic-version": "2023-06-01",
    },
    body: JSON.stringify({
      model: MODEL,
      max_tokens: 1024,
      system: buildSystemPrompt(today, time, loggedIn),
      tools: LEO_TOOLS,
      messages,
    }),
  });
  if (!res.ok) throw new Error(`Claude API ${res.status}`);
  return (await res.json()) as { content: Block[] };
}

async function runServerTool(call: ToolUse, guestId?: string) {
  try {
    const i = call.input;
    let result: unknown;
    if (call.name === "search_rooms") {
      result = await searchRooms(
        {
          query: i.query as string | undefined,
          max_price_usd: i.max_price_usd as number | undefined,
          amenities: i.amenities as string[] | undefined,
          sort: i.sort as "rating" | "price_asc" | "price_desc" | undefined,
        },
        guestId
      );
    } else if (call.name === "get_room_slots") {
      result = await getRoomSlots(String(i.room_id), i.date ? String(i.date) : undefined);
    } else {
      throw new Error("Outil inconnu");
    }
    return { type: "tool_result", tool_use_id: call.id, content: JSON.stringify(result) };
  } catch (e) {
    const message = e instanceof Error ? e.message : "Erreur";
    return { type: "tool_result", tool_use_id: call.id, content: message, is_error: true };
  }
}

export async function POST(req: Request) {
  if (!process.env.ANTHROPIC_API_KEY) {
    return NextResponse.json({ error: "Léo n'est pas configuré." }, { status: 500 });
  }

  let messages: Msg[];
  try {
    ({ messages } = await req.json());
    if (!Array.isArray(messages) || messages.length === 0 || messages.length > MAX_MESSAGES) throw new Error();
  } catch {
    return NextResponse.json({ error: "Requête invalide." }, { status: 400 });
  }

  try {
    const session = await getSession();
    const guestId = session?.role === "GUEST" ? session.sub : undefined;

    for (let loop = 0; loop < MAX_LOOPS; loop++) {
      const { content } = await callClaude(messages, !!guestId);
      messages = [...messages, { role: "assistant", content }];

      const toolUses = content.filter((b): b is ToolUse => b.type === "tool_use");
      if (toolUses.length === 0) {
        return NextResponse.json({ messages, pendingClientCalls: [], serverResults: [] });
      }

      const serverCalls = toolUses.filter((t) => SERVER_TOOL_NAMES.has(t.name));
      const clientCalls = toolUses.filter((t) => !SERVER_TOOL_NAMES.has(t.name));
      const serverResults = await Promise.all(serverCalls.map((c) => runServerTool(c, guestId)));

      // Au moins un outil client : le navigateur exécute, puis renvoie tous les résultats ensemble.
      if (clientCalls.length > 0) {
        return NextResponse.json({ messages, pendingClientCalls: clientCalls, serverResults });
      }

      messages = [...messages, { role: "user", content: serverResults }];
    }
    return NextResponse.json({ error: "Léo a mis trop de temps à répondre." }, { status: 504 });
  } catch {
    return NextResponse.json({ error: "Léo est momentanément indisponible." }, { status: 502 });
  }
}
// Prompt système + outils de Léo.
// Outils "serveur" : exécutés dans /api/leo (lecture des données).
// Outils "client" : exécutés dans le navigateur du client (navigation, wallet, réservation),
// ce qui réutilise sa session existante : Léo n'a jamais d'accès plus large que le client.

export const SERVER_TOOL_NAMES = new Set(["search_rooms", "get_room_slots"]);

export function buildSystemPrompt(todayKinshasa: string, nowKinshasa: string, loggedIn: boolean) {
  return `Tu es Léo, l'assistant de réservation de KinPause, application de chambres d'hôtel en day-use à Kinshasa.

# Personnalité
- Ultra respectueux, courtois et serviable. Tu vouvoies toujours le client.
- Tu dis "Monsieur" ou "Madame" seulement si tu connais le genre du client ; sinon reste neutre et poli.
- Exemple de ton : "Bonjour, je m'occupe de tout pour votre séjour."
- Réponses courtes : 1 à 3 phrases. Le client est souvent fatigué ou pressé.

# Ce que fait KinPause
- On réserve une chambre en journée, à l'heure ou à la journée, sans nuitée. Les prix sont en dollars par créneau.
- Le client paie en ligne un tiers du prix de la chambre (acompte) plus les extras. Le solde est réglé à l'hôtel à l'arrivée.
- Paiement : wallet KinPause ou Mobile Money (Airtel, Orange, M-Pesa).
- Annulation gratuite jusqu'à 30 minutes avant le début du créneau.
- Certains hôtels exigent une pièce d'identité et un âge minimum : signale-le si tu le sais.

# Règles
- Ne jamais inventer une chambre, un prix, un créneau ou une disponibilité. Utilise toujours les outils.
- search_rooms trouve des chambres mais ne connaît PAS les disponibilités : le prix renvoyé est celui du créneau le moins cher. Avant de proposer une chambre pour une date, vérifie ses créneaux avec get_room_slots et ne propose que ceux où isAvailable est vrai.
- Pour search_rooms, passe un seul mot-clé de lieu dans query (ex. "Gombe"), jamais une phrase entière.
- Pour montrer quelque chose au client, utilise navigate (ex. ouvrir la fiche d'une chambre ou la liste filtrée).
- Pour réserver : appelle book_room. Le client verra un récapitulatif et devra confirmer lui-même ; tu ne peux pas contourner cette étape. Avant l'appel, vérifie que tu as la chambre, le créneau, la date et le mode de paiement. Si Mobile Money, demande les 9 chiffres du numéro.
- Pose au maximum une question à la fois. Si une information manque, propose un choix par défaut raisonnable.
- Si un outil échoue ou si un créneau est pris, explique-le simplement et propose une alternative.
- Si on te demande autre chose que KinPause, redirige poliment.

# Contexte
- Date du jour à Kinshasa : ${todayKinshasa}. Heure actuelle : ${nowKinshasa}.
- Client connecté : ${loggedIn ? "oui" : "non. Pour réserver ou voir son wallet, il doit d'abord se connecter : ouvre /login avec navigate et explique-le simplement."}
- Pages de l'application : / (accueil), /explore (liste et filtres), /rooms/<id> (fiche chambre), /map, /concept, /bookings.`;
}

export const LEO_TOOLS = [
  {
    name: "search_rooms",
    description:
      "Cherche des chambres du catalogue. Renvoie id, nom, hôtel, quartier, prix du créneau le moins cher (USD), note et équipements. Ne dit rien de la disponibilité.",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "Un mot-clé : quartier, commune ou nom d'hôtel, ex: Gombe" },
        max_price_usd: { type: "number", description: "Budget maximum par créneau en USD" },
        amenities: {
          type: "array",
          items: { type: "string" },
          description: "Équipements souhaités, ex: piscine, wifi, climatisation",
        },
        sort: { type: "string", enum: ["rating", "price_asc", "price_desc"], description: "Tri (par défaut : meilleures notes)" },
      },
    },
  },
  {
    name: "get_room_slots",
    description: "Liste les créneaux d'une chambre pour une date, avec prix et disponibilité.",
    input_schema: {
      type: "object",
      properties: {
        room_id: { type: "string" },
        date: { type: "string", description: "YYYY-MM-DD (par défaut aujourd'hui)" },
      },
      required: ["room_id"],
    },
  },
  {
    name: "navigate",
    description:
      "Ouvre une page de l'application pour le client. Ex: /explore?need=Travail, /explore?q=Gombe, /rooms/<id>, /bookings.",
    input_schema: {
      type: "object",
      properties: { path: { type: "string", description: "Chemin relatif commençant par /" } },
      required: ["path"],
    },
  },
  {
    name: "get_wallet_balance",
    description: "Renvoie le solde du wallet KinPause du client, en USD.",
    input_schema: { type: "object", properties: {} },
  },
  {
    name: "book_room",
    description:
      "Prépare la réservation et le paiement de l'acompte. Le client voit un récapitulatif et doit confirmer avant tout débit.",
    input_schema: {
      type: "object",
      properties: {
        room_id: { type: "string" },
        slot_id: { type: "string" },
        date: { type: "string", description: "YYYY-MM-DD" },
        payment_method: { type: "string", enum: ["wallet", "mobile_money"] },
        phone: { type: "string", description: "9 chiffres après +243, seulement pour mobile_money" },
        recap: {
          type: "string",
          description: "Récapitulatif lisible : chambre, hôtel, date, horaires, acompte à payer, solde à l'hôtel",
        },
      },
      required: ["room_id", "slot_id", "date", "payment_method", "recap"],
    },
  },
] as const;
"use client";

import { useEffect, useRef, useState } from "react";
import { useRouter } from "next/navigation";

type Block = { type: string; text?: string; [k: string]: unknown };
type Msg = { role: "user" | "assistant"; content: string | Block[] };
type Call = { id: string; name: string; input: Record<string, unknown> };
type BookArgs = {
  room_id: string;
  slot_id: string;
  date: string;
  payment_method: "wallet" | "mobile_money";
  phone?: string;
  recap: string;
};
type ToolOut = { content: string; is_error?: boolean };

const STORAGE_KEY = "leo-chat-v1";
const GREETING = "Bonjour, je suis Léo. Dites-moi ce dont vous avez besoin, je m'occupe du reste.";

// Léo ne peut ouvrir que les pages publiques de l'application.
const PATH_OK = /^\/(?:$|\?|(?:explore|concept|map|bookings)(?:[/?].*)?$|rooms\/[A-Za-z0-9_-]+(?:\?.*)?$)/;

function textOf(m: Msg) {
  if (typeof m.content === "string") return m.content;
  return m.content
    .filter((b) => b.type === "text")
    .map((b) => b.text ?? "")
    .join("\n")
    .trim();
}

export function LeoWidget() {
  const router = useRouter();
  const [open, setOpen] = useState(false);
  const [messages, setMessages] = useState<Msg[]>([]);
  const [input, setInput] = useState("");
  const [busy, setBusy] = useState(false);
  const [notice, setNotice] = useState<string | null>(null);
  const [pending, setPending] = useState<{ args: BookArgs; resolve: (ok: boolean) => void } | null>(null);
  const listRef = useRef<HTMLDivElement>(null);

  // La conversation survit à une redirection (ex. page de paiement Mobile Money).
  useEffect(() => {
    try {
      const saved = sessionStorage.getItem(STORAGE_KEY);
      if (saved) setMessages(JSON.parse(saved));
    } catch {}
  }, []);
  useEffect(() => {
    try {
      sessionStorage.setItem(STORAGE_KEY, JSON.stringify(messages.slice(-30)));
    } catch {}
  }, [messages]);
  useEffect(() => {
    listRef.current?.scrollTo({ top: listRef.current.scrollHeight });
  }, [messages, busy, pending, open]);

  function askConfirmation(args: BookArgs) {
    return new Promise<boolean>((resolve) => setPending({ args, resolve }));
  }
  function answerConfirmation(ok: boolean) {
    pending?.resolve(ok);
    setPending(null);
  }

  async function executeBooking(a: BookArgs): Promise<ToolOut> {
    const json = { "Content-Type": "application/json" };
    const bookingRes = await fetch("/api/bookings", {
      method: "POST",
      headers: json,
      body: JSON.stringify({ roomId: a.room_id, slotId: a.slot_id, date: a.date, extras: [] }),
    });
    const booking = await bookingRes.json().catch(() => ({}));
    if (!bookingRes.ok) {
      const msg = bookingRes.status === 409 ? "Ce créneau vient d'être pris." : booking.error || "Réservation impossible.";
      return { content: msg, is_error: true };
    }
    const id: string = booking.booking.id;

    if (a.payment_method === "wallet") {
      const res = await fetch(`/api/bookings/${id}/pay-with-wallet`, { method: "POST" });
      const data = await res.json().catch(() => ({}));
      if (!res.ok) return { content: data.error || "Paiement par wallet impossible.", is_error: true };
      router.push(data.redirectUrl || `/bookings/${id}`);
      return { content: `Réservation confirmée et payée avec le wallet. Numéro : ${id}.` };
    }

    const digits = String(a.phone ?? "").replace(/\D/g, "").replace(/^243/, "").replace(/^0/, "");
    if (!/^\d{9}$/.test(digits)) return { content: "Numéro Mobile Money invalide : 9 chiffres attendus.", is_error: true };
    const res = await fetch(`/api/bookings/${id}/pay`, {
      method: "POST",
      headers: json,
      body: JSON.stringify({ phone: `+243${digits}`, currency: "USD" }),
    });
    const data = await res.json().catch(() => ({}));
    if (!res.ok) return { content: data.error || "Paiement Mobile Money impossible.", is_error: true };
    if (data.simulated) {
      router.push(data.redirectUrl || `/bookings/${id}`);
      return { content: `Réservation confirmée. Numéro : ${id}.` };
    }
    // Le client est redirigé vers la page de paiement : on laisse à Léo le temps de répondre.
    setTimeout(() => (window.location.href = data.paymentUrl), 1800);
    return { content: "Demande de paiement envoyée. Le client va être redirigé vers la page de paiement Mobile Money pour valider." };
  }

  async function runClientTool(call: Call): Promise<ToolOut> {
    try {
      if (call.name === "navigate") {
        const path = String(call.input.path ?? "");
        if (!PATH_OK.test(path)) return { content: "Chemin non autorisé.", is_error: true };
        router.push(path);
        return { content: `Page ouverte : ${path}` };
      }
      if (call.name === "get_wallet_balance") {
        const res = await fetch("/api/wallet/balance");
        if (!res.ok) return { content: "Solde indisponible (client peut-être non connecté).", is_error: true };
        const data = await res.json();
        return { content: `Solde du wallet : ${data.balance ?? 0} USD` };
      }
      if (call.name === "book_room") {
        const ok = await askConfirmation(call.input as unknown as BookArgs);
        if (!ok) return { content: "Le client a refusé la confirmation. Rien n'a été réservé ni débité." };
        return await executeBooking(call.input as unknown as BookArgs);
      }
      return { content: "Outil inconnu.", is_error: true };
    } catch {
      return { content: "Action impossible pour le moment.", is_error: true };
    }
  }

  async function send(text: string) {
    const clean = text.trim();
    if (!clean || busy) return;
    let msgs: Msg[] = [...messages, { role: "user", content: clean }];
    setMessages(msgs);
    setInput("");
    setNotice(null);
    setBusy(true);
    try {
      for (let step = 0; step < 8; step++) {
        const res = await fetch("/api/leo", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ messages: msgs }),
        });
        const data = await res.json().catch(() => ({}));
        if (!res.ok) throw new Error(data.error || "Léo est momentanément indisponible.");
        msgs = data.messages;
        setMessages(msgs);
        const calls: Call[] = data.pendingClientCalls ?? [];
        if (calls.length === 0) break;

        const clientResults: Block[] = [];
        for (const call of calls) {
          const out = await runClientTool(call);
          clientResults.push({ type: "tool_result", tool_use_id: call.id, ...out });
        }
        msgs = [...msgs, { role: "user", content: [...(data.serverResults ?? []), ...clientResults] }];
        setMessages(msgs);
      }
    } catch (e) {
      setNotice(e instanceof Error ? e.message : "Léo est momentanément indisponible.");
    } finally {
      setBusy(false);
    }
  }

  const visible = messages
    .map((m) => ({ role: m.role, text: textOf(m) }))
    .filter((m) => m.text !== "");

  return (
    <>
      {!open && (
        <button
          type="button"
          onClick={() => setOpen(true)}
          aria-label="Ouvrir Léo, votre assistant"
          className="fixed right-4 bottom-[calc(96px+env(safe-area-inset-bottom))] z-40 h-12 px-5 rounded-full bg-[var(--text-primary)] text-[var(--bg-primary)] text-[14px] font-bold shadow-[0_8px_24px_-8px_rgba(0,0,0,0.4)]"
        >
          Léo
        </button>
      )}

      {open && (
        <section
          role="dialog"
          aria-label="Léo, assistant KinPause"
          className="fixed z-50 inset-x-0 bottom-0 h-[46dvh] md:inset-x-auto md:right-4 md:bottom-4 md:w-[380px] md:h-[560px] flex flex-col bg-[var(--bg-primary)] text-[var(--text-primary)] rounded-t-2xl md:rounded-2xl shadow-[0_-8px_30px_-12px_rgba(0,0,0,0.3)] border border-[var(--border-subtle)]"
        >
          <header className="flex items-center justify-between px-4 pt-3 pb-2 border-b border-[var(--border-subtle)]">
            <p className="text-[15px] font-bold">Léo</p>
            <button
              type="button"
              onClick={() => setOpen(false)}
              aria-label="Réduire Léo"
              className="w-9 h-9 flex items-center justify-center text-[22px] leading-none text-[var(--text-secondary)]"
            >
              ×
            </button>
          </header>

          <div ref={listRef} aria-live="polite" className="flex-1 overflow-y-auto px-4 py-3 flex flex-col gap-2.5">
            <p className="self-start max-w-[85%] rounded-2xl rounded-bl-md bg-[var(--bg-card)] px-3.5 py-2.5 text-[14px] leading-relaxed">
              {GREETING}
            </p>
            {visible.map((m, i) => (
              <p
                key={i}
                className={`max-w-[85%] px-3.5 py-2.5 text-[14px] leading-relaxed whitespace-pre-wrap ${
                  m.role === "user"
                    ? "self-end rounded-2xl rounded-br-md bg-[var(--text-primary)] text-[var(--bg-primary)]"
                    : "self-start rounded-2xl rounded-bl-md bg-[var(--bg-card)]"
                }`}
              >
                {m.text}
              </p>
            ))}
            {busy && !pending && <p className="self-start text-[12px] text-[var(--text-secondary)]">Léo s'en occupe…</p>}

            {pending && (
              <div className="self-stretch rounded-2xl border border-[var(--border-medium)] bg-[var(--bg-card)] p-4">
                <p className="text-[13px] font-bold mb-1.5">Confirmez-vous cette réservation ?</p>
                <p className="text-[13px] leading-relaxed text-[var(--text-secondary)] whitespace-pre-wrap">
                  {pending.args.recap}
                </p>
                <div className="flex gap-2 mt-3">
                  <button
                    type="button"
                    onClick={() => answerConfirmation(true)}
                    className="flex-1 h-11 rounded-xl bg-[var(--text-primary)] text-[var(--bg-primary)] text-[14px] font-bold"
                  >
                    Confirmer et payer
                  </button>
                  <button
                    type="button"
                    onClick={() => answerConfirmation(false)}
                    className="h-11 px-4 rounded-xl bg-[var(--bg-secondary)] text-[14px] font-semibold"
                  >
                    Annuler
                  </button>
                </div>
              </div>
            )}

            {notice && (
              <p role="alert" className="text-[12.5px] text-[var(--error)]">
                {notice}
              </p>
            )}
          </div>

          <div className="flex items-center gap-2 px-3 pt-2 pb-[max(12px,env(safe-area-inset-bottom))] border-t border-[var(--border-subtle)]">
            <label htmlFor="leo-input" className="sr-only">
              Votre message à Léo
            </label>
            <input
              id="leo-input"
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && send(input)}
              placeholder="Ex. une chambre calme à Gombe ce soir"
              disabled={busy}
              className="flex-1 min-w-0 h-11 rounded-xl bg-[var(--bg-card)] border border-[var(--border-medium)] px-3.5 text-[16px] outline-none"
            />
            <button
              type="button"
              onClick={() => send(input)}
              disabled={busy || !input.trim()}
              className="h-11 px-4 rounded-xl bg-[var(--text-primary)] text-[var(--bg-primary)] text-[14px] font-bold disabled:opacity-40"
            >
              Envoyer
            </button>
          </div>
        </section>
      )}
    </>
  );
}
