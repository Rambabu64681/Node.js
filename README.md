# Node.js
/**
 * Minimal FHIR Patient API (Node.js + Express + MongoDB)
 * - POST /fhir/Patient       -> create a Patient resource
 * - GET  /fhir/Patient/:id   -> fetch a Patient resource by id
 *
 * NOTE: This is a simplified example for learning and internal prototypes.
 * In real healthcare production, add auth (OAuth2/SMART), audit logs, encryption, access control, etc.
 */

const express = require("express");
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");
const mongoose = require("mongoose");

const app = express();
app.use(express.json({ limit: "1mb" }));

// Basic security hardening (recommended baseline)
app.use(helmet());

// Basic abuse protection (helpful for public-facing APIs)
app.use(
  rateLimit({
    windowMs: 60 * 1000,
    max: 60, // 60 req/min per IP
  })
);

// ---- MongoDB connection ----
const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017/healthcare_fhir";
mongoose
  .connect(MONGO_URI)
  .then(() => console.log("MongoDB connected"))
  .catch((e) => {
    console.error("MongoDB connection error:", e.message);
    process.exit(1);
  });

// ---- FHIR Patient schema (simplified) ----
// FHIR Patient: https://hl7.org/fhir/patient.html
const PatientSchema = new mongoose.Schema(
  {
    resourceType: { type: String, required: true, enum: ["Patient"] },
    active: { type: Boolean, default: true },
    name: [
      {
        family: { type: String, required: true },
        given: [{ type: String, required: true }],
      },
    ],
    gender: { type: String, enum: ["male", "female", "other", "unknown"] },
    birthDate: { type: String }, // YYYY-MM-DD (keep as string for simplicity)
    telecom: [
      {
        system: { type: String, enum: ["phone", "email"] },
        value: { type: String },
      },
    ],
  },
  { timestamps: true }
);

const Patient = mongoose.model("Patient", PatientSchema);

// ---- Lightweight FHIR validation ----
function validateFHIRPatient(body) {
  if (!body || body.resourceType !== "Patient") return "resourceType must be 'Patient'";
  if (!Array.isArray(body.name) || body.name.length === 0) return "name[] is required";
  if (!body.name[0]?.family) return "name[0].family is required";
  if (!Array.isArray(body.name[0]?.given) || body.name[0].given.length === 0)
    return "name[0].given[] is required";
  if (body.gender && !["male", "female", "other", "unknown"].includes(body.gender))
    return "gender must be one of male|female|other|unknown";
  if (body.birthDate && !/^\d{4}-\d{2}-\d{2}$/.test(body.birthDate))
    return "birthDate must be YYYY-MM-DD";
  return null;
}

// ---- Routes ----

// Create a Patient (FHIR style)
app.post("/fhir/Patient", async (req, res) => {
  try {
    const err = validateFHIRPatient(req.body);
    if (err) return res.status(400).json({ error: err });

    const saved = await Patient.create(req.body);

    // FHIR-ish response headers
    res
      .status(201)
      .set("Location", `/fhir/Patient/${saved._id}`)
      .json({
        resourceType: "Patient",
        id: String(saved._id),
        ...req.body,
      });
  } catch (e) {
    res.status(500).json({ error: "Internal server error" });
  }
});

// Read a Patient by id (FHIR style)
app.get("/fhir/Patient/:id", async (req, res) => {
  try {
    const patient = await Patient.findById(req.params.id).lean();
    if (!patient) return res.status(404).json({ error: "Patient not found" });

    // Return a proper FHIR Patient resource with "id"
    const { _id, __v, createdAt, updatedAt, ...rest } = patient;
    res.json({
      resourceType: "Patient",
      id: String(_id),
      ...rest,
    });
  } catch (e) {
    res.status(400).json({ error: "Invalid id" });
  }
});

// Simple health check endpoint
app.get("/health", (_, res) => res.json({ status: "ok" }));

// ---- Start server ----
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`FHIR API running on http://localhost:${PORT}`));

/**
 * Quick run:
 * 1) npm init -y
 * 2) npm i express mongoose helmet express-rate-limit
 * 3) (optional) export MONGO_URI="mongodb://localhost:27017/healthcare_fhir"
 * 4) node server.js
 *
 * Test:
 * curl -X POST http://localhost:3000/fhir/Patient \
 *  -H "Content-Type: application/json" \
 *  -d '{"resourceType":"Patient","active":true,"name":[{"family":"Doe","given":["Jane"]}],"gender":"female","birthDate":"1990-02-14","telecom":[{"system":"phone","value":"+15551234567"}]}'
 */
