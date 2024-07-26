from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy
import logging

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://postgres:Vikasdatabase@localhost:5432/User_management_db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    company_name = db.Column(db.String(100))
    age = db.Column(db.Integer)
    city = db.Column(db.String(50))
    state = db.Column(db.String(30))
    zip = db.Column(db.Integer)
    email = db.Column(db.String(100), unique=True)
    web = db.Column(db.String(255))

with app.app_context():
    db.create_all()

# Set up logging
logging.basicConfig(level=logging.DEBUG,
                    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    handlers=[
                        logging.FileHandler("app.log"),
                        logging.StreamHandler()
                    ])

@app.route('/api/users', methods=['GET'])
def get_users():
    page = int(request.args.get('page', 1))
    limit = int(request.args.get('limit', 5))
    search_term = request.args.get('search', None)
    sort_field = request.args.get('sort', None)

    query = User.query

    if search_term:
        app.logger.debug(f'Searching for users with term: {search_term}')
        query = query.filter(db.or_(
            User.first_name.ilike(f"%{search_term}%"),
            User.last_name.ilike(f"%{search_term}%")
        ))

    if sort_field:
        app.logger.debug(f'Sorting users by: {sort_field}')
        order = sort_field if not sort_field.startswith('-') else sort_field[1:]
        query = query.order_by(getattr(User, order).asc() if sort_field.startswith('-') else getattr(User, order).desc())

    pagination = query.paginate(page=page, per_page=limit, error_out=False)
    users = pagination.items

    user_list = [
        {'id': user.id, 'first_name': user.first_name, 'last_name': user.last_name,
         'company_name': user.company_name, 'age': user.age, 'city': user.city,
         'state': user.state, 'zip': user.zip, 'email': user.email,
         'web': user.web} for user in users
    ]

    app.logger.info(f'Fetched {len(user_list)} users')
    return jsonify({
        "users": user_list,
        "total": pagination.total,
        "pages": pagination.pages,
        "current_page": page
    }), 200

@app.route('/api/user', methods=['POST'])
def create_user():
    data = request.get_json()
    if not data:
        app.logger.warning('No input data provided for user creation')
        return jsonify({'message': 'No input data provided'}), 400

    try:
        if isinstance(data, list):
            users = [
                User(
                    first_name=user['first_name'],
                    last_name=user['last_name'],
                    company_name=user.get('company_name'),
                    age=user.get('age'),
                    city=user.get('city'),
                    state=user.get('state'),
                    zip=user.get('zip'),
                    email=user['email'],
                    web=user.get('web')
                ) for user in data
            ]
            db.session.add_all(users)
        else:
            new_user = User(
                first_name=data['first_name'],
                last_name=data['last_name'],
                company_name=data.get('company_name'),
                age=data.get('age'),
                city=data.get('city'),
                state=data.get('state'),
                zip=data.get('zip'),
                email=data['email'],
                web=data.get('web')
            )
            db.session.add(new_user)

        db.session.commit()
        app.logger.info('Users created successfully')
        return jsonify({'message': 'Users created successfully'}), 201
    except Exception as e:
        db.session.rollback()
        app.logger.error(f"Error occurred during user creation: {str(e)}")
        return jsonify({'message': 'An error occurred', 'error': str(e)}), 500

@app.route('/api/users/<int:id>', methods=['GET'])
def get_user_by_id(id):
    app.logger.debug(f'Fetching user with id {id}')
    user = User.query.get(id)
    if user:
        user_data = {
            'id': user.id,
            'first_name': user.first_name,
            'last_name': user.last_name,
            'company_name': user.company_name,
            'age': user.age,
            'city': user.city,
            'state': user.state,
            'zip': user.zip,
            'email': user.email,
            'web': user.web
        }
        app.logger.info(f'User with id {id} fetched successfully')
        return jsonify(user_data), 200
    else:
        app.logger.warning(f'User with id {id} not found')
        return jsonify({'message': 'User not found'}), 404

@app.route('/api/users/<int:id>', methods=['PUT'])
def update_user(id):
    data = request.get_json()
    user = User.query.get(id)
    if not user:
        app.logger.warning(f'User with id {id} not found for update')
        return jsonify({'message': 'User not found'}), 404

    try:
        if 'first_name' in data:
            user.first_name = data['first_name']
        if 'last_name' in data:
            user.last_name = data['last_name']
        if 'age' in data:
            user.age = data['age']

        db.session.commit()
        app.logger.info(f'User with id {id} updated successfully')
        return jsonify({'message': 'User updated successfully'}), 200
    except Exception as e:
        db.session.rollback()
        app.logger.error(f"Error occurred during user update: {str(e)}")
        return jsonify({'message': 'An error occurred', 'error': str(e)}), 500

@app.route('/api/users/<int:id>', methods=['DELETE'])
def delete_user(id):
    user = User.query.get(id)
    if not user:
        app.logger.warning(f'User with id {id} not found for deletion')
        return jsonify({'message': 'User not found'}), 404

    try:
        db.session.delete(user)
        db.session.commit()
        app.logger.info(f'User with id {id} deleted successfully')
        return '', 204
    except Exception as e:
        db.session.rollback()
        app.logger.error(f"Error occurred during user deletion: {str(e)}")
        return jsonify({'message': 'An error occurred', 'error': str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True)
