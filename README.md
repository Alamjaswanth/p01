# p01
Implement API endpoints

  # Import Flask and SQLAlchemy
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

# Create a Flask app and a database
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///appointments.db'
db = SQLAlchemy(app)

# Define a model for doctors
class Doctor(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(80), nullable=False)
    specialty = db.Column(db.String(80), nullable=False)
    max_patients = db.Column(db.Integer, nullable=False)
    schedule = db.Column(db.String(7), nullable=False) # A string of 0s and 1s representing availability on Mon-Sat

    def __repr__(self):
        return f'<Doctor {self.name}>'

    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'specialty': self.specialty,
            'max_patients': self.max_patients,
            'schedule': self.schedule
        }

# Define a model for appointments
class Appointment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    doctor_id = db.Column(db.Integer, db.ForeignKey('doctor.id'), nullable=False)
    patient_name = db.Column(db.String(80), nullable=False)
    date = db.Column(db.Date, nullable=False)

    def __repr__(self):
        return f'<Appointment {self.patient_name} with {self.doctor_id} on {self.date}>'

    def to_dict(self):
        return {
            'id': self.id,
            'doctor_id': self.doctor_id,
            'patient_name': self.patient_name,
            'date': self.date
        }

# Create the database tables
db.create_all()

# Define an endpoint for getting all doctors
@app.route('/doctors', methods=['GET'])
def get_doctors():
    doctors = Doctor.query.all()
    return jsonify([doctor.to_dict() for doctor in doctors])

# Define an endpoint for getting a single doctor by id
@app.route('/doctors/<int:id>', methods=['GET'])
def get_doctor(id):
    doctor = Doctor.query.get_or_404(id)
    return jsonify(doctor.to_dict())

# Define an endpoint for booking an appointment with a doctor by id
@app.route('/doctors/<int:id>/book', methods=['POST'])
def book_appointment(id):
    doctor = Doctor.query.get_or_404(id)
    data = request.get_json()
    patient_name = data.get('patient_name')
    date = data.get('date')
    # Validate the input data
    if not patient_name or not date:
        return jsonify({'error': 'Missing patient name or date'}), 400
    # Check if the doctor is available on that date
    weekday = date.weekday()
    if weekday == 6 or doctor.schedule[weekday] == '0':
        return jsonify({'error': 'Doctor is not available on that date'}), 400
    # Check if the doctor has reached the maximum number of patients on that date
    appointments = Appointment.query.filter_by(doctor_id=id, date=date).count()
    if appointments >= doctor.max_patients:
        return jsonify({'error': 'Doctor has no more slots on that date'}), 400
    # Create a new appointment and save it to the database
    appointment = Appointment(doctor_id=id, patient_name=patient_name, date=date)
    db.session.add(appointment)
    db.session.commit()
    return jsonify(appointment.to_dict()), 201

# Run the app
if __name__ == '__main__':
    app.run(debug=True)
