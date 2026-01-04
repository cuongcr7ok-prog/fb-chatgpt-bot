import express from "express";
import axios from "axios";
import OpenAI from "openai";

const app = express();
app.use(express.json());

const { FB_VERIFY_TOKEN, FB_PAGE_ACCESS_TOKEN, OPENAI_API_KEY } = process.env;

const openai = new OpenAI({ apiKey: OPENAI_API_KEY });
const memory = new Map();

app.get("/webhook", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === FB_VERIFY_TOKEN) return res.status(200).send(challenge);
  return res.sendStatus(403);
});

app.post("/webhook", async (req, res) => {
  try {
    const body = req.body;
    if (body.object !== "page") return res.sendStatus(404);

    for (const entry of body.entry || []) {
      for (const event of entry.messaging || []) {
        const psid = event.sender?.id;
        const text = event.message?.text;
        if (!psid || !text) continue;

        const reply = await getBotReply(psid, text);
        await sendText(psid, reply);
      }
    }
    return res.status(200).send("EVENT_RECEIVED");
  } catch (e) {
    console.error(e);
    return res.status(200).send("OK");
  }
});

async function getBotReply(psid, userText) {
  const history = memory.get(psid) || [];
  const system = `
Bạn là chatbot CSKH cho shop thời trang.
- Trả lời ngắn gọn, lịch sự, hỏi thêm thông tin khi thiếu.
- Đổi/trả trong 7 ngày kể từ khi nhận hàng nếu còn tem/mác, chưa giặt/chưa sử dụng.
- Khi khách bức xúc: xin lỗi + xin mã đơn/sđt + hướng dẫn xử lý + chuyển CSKH nếu cần.
`;

  const input = [{ role: "system", content: system }, ...history.slice(-12), { role: "user", content: userText }];

  const r = await openai.responses.create({
    model: "gpt-4.1-mini",
    input
  });

  const out = r.output_text || "Dạ anh/chị nói lại giúp em rõ hơn được không ạ?";
  memory.set(psid, [...history, { role: "user", content: userText }, { role: "assistant", content: out }]);
  return out;
}

async function sendText(psid, text) {
  await axios.post(
    "https://graph.facebook.com/v20.0/me/messages",
    { recipient: { id: psid }, message: { text } },
    { params: { access_token: FB_PAGE_ACCESS_TOKEN } }
  );
}

const port = process.env.PORT || 10000;
app.listen(port, () => console.log("Listening on", port));
