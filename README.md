// Cloudflare Worker — SFA Toner Mailer
// Paste this into your Cloudflare Worker editor and click "Deploy"
//
// SETUP:
// 1. In Cloudflare Worker settings -> Variables -> add a secret named RESEND_API_KEY
//    with your Resend API key as the value (keep "Encrypt" checked).
// 2. Update FROM_EMAIL and ALLOWED_ORIGIN below.
// 3. Deploy. You'll get a URL like https://sfa-mailer.yourname.workers.dev
// 4. Put that URL into the SFA app's WORKER_URL constant.

const FROM_EMAIL = "Toner Alerts <onboarding@resend.dev>"; // change once your domain is verified in Resend
const ALLOWED_ORIGIN = "*"; // for production, replace * with your app's exact URL

export default {
  async fetch(request, env) {
    // Handle CORS preflight
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": ALLOWED_ORIGIN,
          "Access-Control-Allow-Methods": "POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type",
        },
      });
    }

    if (request.method !== "POST") {
      return new Response("Method not allowed", { status: 405 });
    }

    try {
      const body = await request.json();
      const { to, subject, text } = body;

      if (!to || !subject || !text) {
        return jsonResponse({ error: "Missing to, subject, or text" }, 400);
      }

      const resendResp = await fetch("https://api.resend.com/emails", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${env.RESEND_API_KEY}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          from: FROM_EMAIL,
          to: [to],
          subject: subject,
          text: text,
        }),
      });

      const result = await resendResp.json();

      if (!resendResp.ok) {
        return jsonResponse({ error: "Resend API error", details: result }, 502);
      }

      return jsonResponse({ success: true, id: result.id });
    } catch (err) {
      return jsonResponse({ error: err.message }, 500);
    }
  },
};

function jsonResponse(obj, status = 200) {
  return new Response(JSON.stringify(obj), {
    status,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": ALLOWED_ORIGIN,
    },
  });
}# stanfab-app
