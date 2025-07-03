// Estrutura inicial do backend completo para site de apostas

// server/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const dotenv = require('dotenv');

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

// Conexão MongoDB
mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log('MongoDB conectado'))
  .catch((err) => console.error(err));

// Rotas
app.use('/api/auth', require('./routes/auth'));
app.use('/api/games', require('./routes/games'));
app.use('/api/bets', require('./routes/bets'));

// Iniciar servidor
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));


// server/models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
  balance: { type: Number, default: 100 },
});

module.exports = mongoose.model('User', UserSchema);


// server/models/Game.js
const mongoose = require('mongoose');

const GameSchema = new mongoose.Schema({
  teamA: String,
  teamB: String,
  odds: { teamA: Number, draw: Number, teamB: Number },
  date: Date,
});

module.exports = mongoose.model('Game', GameSchema);


// server/models/Bet.js
const mongoose = require('mongoose');

const BetSchema = new mongoose.Schema({
  userId: mongoose.Schema.Types.ObjectId,
  gameId: mongoose.Schema.Types.ObjectId,
  choice: String,
  amount: Number,
  odds: Number,
  status: { type: String, default: 'pending' },
});

module.exports = mongoose.model('Bet', BetSchema);


// server/routes/auth.js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

router.post('/register', async (req, res) => {
  const { name, email, password } = req.body;
  const hashed = await bcrypt.hash(password, 10);
  try {
    const user = await User.create({ name, email, password: hashed });
    res.json(user);
  } catch (err) {
    res.status(400).json({ error: 'Usuário já existe' });
  }
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ error: 'Usuário não encontrado' });

  const match = await bcrypt.compare(password, user.password);
  if (!match) return res.status(401).json({ error: 'Senha incorreta' });

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
  res.json({ token, user });
});

module.exports = router;


// server/routes/games.js
const express = require('express');
const Game = require('../models/Game');
const router = express.Router();

router.get('/', async (req, res) => {
  const games = await Game.find();
  res.json(games);
});

router.post('/', async (req, res) => {
  const game = await Game.create(req.body);
  res.json(game);
});

module.exports = router;


// server/routes/bets.js
const express = require('express');
const Bet = require('../models/Bet');
const User = require('../models/User');
const router = express.Router();

router.post('/', async (req, res) => {
  const { userId, gameId, choice, amount, odds } = req.body;
  const user = await User.findById(userId);
  if (user.balance < amount) return res.status(400).json({ error: 'Saldo insuficiente' });

  user.balance -= amount;
  await user.save();

  const bet = await Bet.create({ userId, gameId, choice, amount, odds });
  res.json(bet);
});

router.get('/:userId', async (req, res) => {
  const bets = await Bet.find({ userId: req.params.userId });
  res.json(bets);
});

module.exports = router;
