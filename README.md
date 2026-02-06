import express from "express";
import multer from "multer";
import cors from "cors";
import fetch from "node-fetch";
import cloudinary from "cloudinary";
import fs from "fs";

const app = express();
app.use(cors());
app.use(express.json());

const upload = multer({ dest: "uploads/" });

// Cloudinary configuration
cloudinary.v2.config({
  cloud_name: process.env.CLOUD_NAME,   // Cloudinary ka name
  api_key: process.env.CLOUD_KEY,       // Cloudinary API key
  api_secret: process.env.CLOUD_SECRET  // Cloudinary secret
});

const SHOTSTACK_KEY = process.env.SHOTSTACK_KEY;  // Shotstack API key
const SHOTSTACK_URL = "https://api.shotstack.io/stage/render";

// Upload video + AI edit
app.post("/upload", upload.single("video"), async (req, res) => {
  try {
    // 1️⃣ Upload video to Cloudinary
    const uploadRes = await cloudinary.v2.uploader.upload(
      req.file.path,
      { resource_type: "video" }
    );
    fs.unlinkSync(req.file.path); // temp file delete

    const videoUrl = uploadRes.secure_url;

    // 2️⃣ Send video to Shotstack for AI editing
    const body = {
      timeline: {
        tracks: [
          {
            clips: [
              {
                asset: {
                  type: "video",
                  src: videoUrl
                },
                start: 0,
                length: 8
              }
            ]
          }
        ]
      },
      output: {
        format: "mp4",
        resolution: "1080x1920"
      }
    };

    const response = await fetch(SHOTSTACK_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-api-key": SHOTSTACK_KEY
      },
      body: JSON.stringify(body)
    });

    const data = await response.json();

    res.json({ renderId: data.response.id });
  } catch (e) {
    res.status(500).json({ error: "Video processing failed" });
  }
});

// Check render status
app.get("/status/:id", async (req, res) => {
  const response = await fetch(
    `https://api.shotstack.io/stage/render/${req.params.id}`,
    { headers: { "x-api-key": SHOTSTACK_KEY } }
  );
  const data = await response.json();
  res.json(data.response);
});

app.listen(3000, () => console.log("AI Video Server Running"));
