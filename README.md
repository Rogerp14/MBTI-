from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dating_app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Define User Model
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    mbti_type = db.Column(db.String(4), nullable=False)
    bio = db.Column(db.Text, nullable=True)

# Compatibility Function (based on MBTI types)
COMPATIBILITY = {
    "INTJ": ["ENFP", "ENTP"],
    "INFP": ["ENFJ", "ENTJ"],
    "ENTJ": ["INFP", "ISFP"],
    "ENFP": ["INTJ", "INFJ"],
    # Continue for all 16 types
}

def calculate_compatibility(user_mbti, other_mbti):
    score = 0

    # Check for compatibility in the first letter (I/E)
    if user_mbti[0] == other_mbti[0]:
        score += 1

    # Check for compatibility in the second letter (S/N)
    if user_mbti[1] == other_mbti[1]:
        score += 1

    # Check for compatibility in the third letter (T/F)
    if user_mbti[2] == other_mbti[2]:
        score += 1

    # Check for compatibility in the fourth letter (J/P)
    if user_mbti[3] == other_mbti[3]:
        score += 1

    # Bonus for highly compatible types from the COMPATIBILITY dictionary
    if other_mbti in COMPATIBILITY.get(user_mbti, []):
        score += 2  # Bonus points for compatibility based on the dictionary

    return score

# Initialize the Database
with app.app_context():
    db.create_all()

# User Registration Route
@app.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    username = data.get('username')
    mbti_type = data.get('mbti_type')
    bio = data.get('bio', '')

    if User.query.filter_by(username=username).first():
        return jsonify({"error": "Username already exists"}), 400

    user = User(username=username, mbti_type=mbti_type, bio=bio)
    db.session.add(user)
    db.session.commit()

    return jsonify({"message": "User registered successfully!"})

# Get Compatible Matches Route with Score
@app.route('/matches/<username>', methods=['GET'])
def get_matches(username):
    user = User.query.filter_by(username=username).first()
    if not user:
        return jsonify({"error": "User not found"}), 404

    potential_matches = User.query.filter(User.username != username).all()
    matches = []

    for match in potential_matches:
        compatibility_score = calculate_compatibility(user.mbti_type, match.mbti_type)
        if compatibility_score > 0:  # Only show users with a positive compatibility score
            matches.append({
                "username": match.username,
                "mbti_type": match.mbti_type,
                "bio": match.bio,
                "compatibility_score": compatibility_score
            })

    # Sort matches by compatibility score (highest first)
    matches.sort(key=lambda x: x["compatibility_score"], reverse=True)

    return jsonify({"matches": matches})

# Send a Message Route (simplified for demo)
@app.route('/message', methods=['POST'])
def send_message():
    data = request.get_json()
    sender = data.get('sender')
    recipient = data.get('recipient')
    message = data.get('message')

    # In a real app, messages would be stored in a database table
    return jsonify({
        "message": "Message sent!",
        "from": sender,
        "to": recipient,
        "content": message
    })

if __name__ == '__main__':
    app.run(debug=True)
