rails new medical_portal -d postgresql
cd medical_portal

default: &default
  adapter: postgresql
  encoding: unicode
  pool: 5

development:
  <<: *default
  database: medical_portal_development

test:
  <<: *default
  database: medical_portal_test

production:
  <<: *default
  database: medical_portal_production
  username: medical_portal
  password: <%= ENV['MEDICAL_PORTAL_DATABASE_PASSWORD'] %>

rails db:create
gem 'devise'
bundle install
rails generate devise:install
rails generate devise User
rails generate migration AddRoleToUsers role:string
class AddRoleToUsers < ActiveRecord::Migration[6.1]
  def change
    add_column :users, :role, :string, default: "receptionist"
  end
end
rails db:migrate
rails generate scaffold Patient name:string age:integer address:text phone:string email:string
rails db:migrate
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  enum role: { receptionist: 'receptionist', doctor: 'doctor' }

  validates :role, presence: true
end
class PatientsController < ApplicationController
  before_action :authenticate_user!
  before_action :authorize_receptionist!, except: [:index, :show]

  # other actions...

  private

  def authorize_receptionist!
    redirect_to root_path, alert: 'Access Denied' unless current_user.receptionist?
  end
end
rails generate controller Doctors index
Rails.application.routes.draw do
  devise_for :users
  resources :patients
  get 'doctors', to: 'doctors#index'
  root 'home#index'
end
class DoctorsController < ApplicationController
  before_action :authenticate_user!
  before_action :authorize_doctor!

  def index
    @patients = Patient.all
    @patient_registration_data = Patient.group_by_day(:created_at).count
  end

  private

  def authorize_doctor!
    redirect_to root_path, alert: 'Access Denied' unless current_user.doctor?
  end
end
<h1>Patients List</h1>
<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Age</th>
      <th>Address</th>
      <th>Phone</th>
      <th>Email</th>
    </tr>
  </thead>
  <tbody>
    <% @patients.each do |patient| %>
      <tr>
        <td><%= patient.name %></td>
        <td><%= patient.age %></td>
        <td><%= patient.address %></td>
        <td><%= patient.phone %></td>
        <td><%= patient.email %></td>
      </tr>
    <% end %>
  </tbody>
</table>

<h2>Patient Registrations Over Time</h2>
<div id="patient-registration-chart"></div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
  document.addEventListener('DOMContentLoaded', function() {
    var ctx = document.getElementById('patient-registration-chart').getContext('2d');
    var chart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: <%= @patient_registration_data.keys %>,
        datasets: [{
          label: 'Number of Patients Registered',
          data: <%= @patient_registration_data.values %>,
          borderColor: 'rgba(75, 192, 192, 1)',
          borderWidth: 2,
          fill: false
        }]
      },
      options: {
        scales: {
          x: {
            type: 'time',
            time: {
              unit: 'day'
            }
          }
        }
      }
    });
  });
</script>
rails console
User.create(email: 'receptionist@example.com', password: 'password', password_confirmation: 'password', role: 'receptionist')
User.create(email: 'doctor@example.com', password: 'password', password_confirmation: 'password', role: 'doctor')
