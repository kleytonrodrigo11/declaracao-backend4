// backend/server.js
const app = require('./app');
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});


// backend/app.js
require('dotenv').config();
const express = require('express');
const multer = require('multer');
const cors = require('cors');
const declaracaoRoutes = require('./routes/declaracaoRoutes');

const app = express();
app.use(cors());
app.use(express.json());
app.use('/uploads', express.static('uploads'));

app.use('/api', declaracaoRoutes);

module.exports = app;


// backend/routes/declaracaoRoutes.js
const express = require('express');
const multer = require('multer');
const {
  criarDeclaracao,
  obterDeclaracao,
  iniciarPagamento,
  webhookPagamento,
  gerarQrCode
} = require('../controllers/declaracaoController');

const router = express.Router();

const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, 'uploads/'),
  filename: (req, file, cb) => cb(null, Date.now() + '-' + file.originalname)
});

const upload = multer({ storage });

router.post('/declaracao', upload.fields([{ name: 'foto' }, { name: 'audio' }]), criarDeclaracao);
router.get('/declaracao/:id', obterDeclaracao);
router.post('/pagamento', iniciarPagamento);
router.post('/webhook-pagamento', webhookPagamento);
router.get('/qr/:id', gerarQrCode);

module.exports = router;


// backend/controllers/declaracaoController.js
const path = require('path');
const qrcode = require('qrcode');
const { criarPreferencia, verificarPagamento } = require('../services/pagamentoService');
const declaracoes = {};

const criarDeclaracao = (req, res) => {
  const id = Date.now().toString();
  const { texto, musica } = req.body;
  const foto = req.files['foto']?.[0]?.filename;
  const audio = req.files['audio']?.[0]?.filename;

  declaracoes[id] = {
    id,
    texto,
    musica,
    foto,
    audio,
    pago: false
  };

  res.json({ id });
};

const obterDeclaracao = (req, res) => {
  const { id } = req.params;
  const declaracao = declaracoes[id];

  if (!declaracao) return res.status(404).send('Não encontrada');
  if (!declaracao.pago) return res.status(403).send('Pagamento pendente');

  res.json(declaracao);
};

const iniciarPagamento = async (req, res) => {
  const { id } = req.body;
  const declaracao = declaracoes[id];
  if (!declaracao) return res.status(404).send('Não encontrada');

  const urlPagamento = await criarPreferencia(id);
  res.json({ urlPagamento });
};

const webhookPagamento = (req, res) => {
  const { data } = req.body;
  const id = data?.metadata?.id;
  if (declaracoes[id]) {
    declaracoes[id].pago = true;
  }
  res.sendStatus(200);
};

const gerarQrCode = async (req, res) => {
  const { id } = req.params;
  const url = `${process.env.FRONTEND_URL}/declaracao/${id}`;
  const qr = await qrcode.toDataURL(url);
  res.json({ qr });
};

module.exports = {
  criarDeclaracao,
  obterDeclaracao,
  iniciarPagamento,
  webhookPagamento,
  gerarQrCode
};


// backend/services/pagamentoService.js
const mercadopago = require('mercadopago');

mercadopago.configure({ access_token: process.env.MERCADO_PAGO_TOKEN });

const criarPreferencia = async (id) => {
  const preference = {
    items: [{ title: 'Página de declaração', quantity: 1, currency_id: 'BRL', unit_price: 10 }],
    metadata: { id },
    notification_url: `${process.env.BACKEND_URL}/api/webhook-pagamento`
  };
  const response = await mercadopago.preferences.create(preference);
  return response.body.init_point;
};

module.exports = { criarPreferencia };
