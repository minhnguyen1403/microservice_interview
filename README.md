# avian-test-interview
--- Build ticket booking microservice using nodejs redis, lua-script, docker swarm, traefix, rabbitmq



#  setup and usage instructions 
# I have deployed it to my DigitalOcean account, and I am also using Cloudflare and my provider to deploy my system.
# At the end of this document, I will provide a complete curl API flow, starting from registration and login to booking and canceling a ticket.
- About information:
    + Domain api: api.minhnguyen.info.vn
    + Database + Redis: You can retrieve data config from the config.json file of each service.
    + For local api: api.localhost
    
#  an overview of your system architecture and folder structure:
- There are five services:

    + avian_auth: Responsible for user registration and login.
    + avian_concerts: Manages concerts, seat types, and the remaining number of seats for each concert.
    + avian_booking: Handles booking creation and cancellation.
    + avian_framework contains shared configuration for all services, such as middleware, logging, and the initialization of database, Redis, and RabbitMQ connections (commonly referred to as infrastructure setup). This service is included in other services via Git submodule.
    + avian_sdk is responsible for interacting with other services. It includes classes like req-builder and req-executor to communicate via HTTP, and it is also included in services using a Git submodule.

- I use Traefik for routing.
- Use the Authorization: Bearer <token> header to access the services. For internal service-to-service communication, the Avian-Access-Trusted key is used.

# this is localhost environment
- I deploy and test on centos
- docker create avian_network
- sudo nano /etc/hosts
- 127.0.0.1 api.localhost mongo.localhost redis.localhost
- cd docker-compose-all-system
- docker compose up -d
- create database for each service:
- docker exec -it <container_id> mongosh -u root -p 123456
- use admin
- db.createUser({ user: "avian_users", pwd: "123465", roles: [{ role: "readWrite", db: "avian_users" }] }) // create db for service avian_users
- db.createUser({ user: "avian_concerts", pwd: "123465", roles: [{ role: "readWrite", db: "avian_concerts" }] }) // create db for service avian_concerts
- db.createUser({ user: "avian_booking", pwd: "123465", roles: [{ role: "readWrite", db: "avian_booking" }] }) // create db for service avian_booking
- if error from db on service, just docker compose up
- you can cd to source service: npm install - npm run dev

# Noted some error:
For Docker versions lower than 20, you cannot use extra_hosts: - "api.localhost:host-gateway" to enable communication between two containers representing different services. Instead, use the following configuration:
extra_hosts:
  - "api.localhost:${DOCKER_HOST_GATEWAY}"
Additionally, export the DOCKER_HOST_GATEWAY by running:
export DOCKER_HOST_GATEWAY=$(ip route | awk '/default/ {print $3}')
  - For macos: api.localhost:host.docker.internal

- Error: EACCES: permission denied, open '/var/log/service-2025-05-13.log'
Emitted 'error' event at: => This error will occur on macOS if you use ./var (relative path) as the log directory.

# Functional Requirements:
1. Users can register and log in.
    - For login and registration within the framework, these two routes will be passed without requiring authentication.

2. API to list all available concerts.
    # Endpoint: GET /v1/concerts?populate_field=seat_types&status=1
    # Description: Returns a list of available concerts, including seat types: remaining_seats.

3. API to view concert details:
    *  All seat types.
    *  Remaining tickets for each seat type.
 # Endpoint: GET /v1/seat-types?concert_id=6821e93f457d2998c4a384e4
        
4. Booking API:
    * Users can book one ticket per concert.
    * Must choose a seat type.
    * Cannot book if tickets are sold out.
  # Endpoint: POST /v1/seat-types/booking
  - Body: {
        "concert_id": "6821e93f457d2998c4a384e4",
        "seat_type": "3"
    } => userId get from jwt
  # Description: Validate whether the selected seat_type still has available seats or is sold out. Then, use a Lua script to handle concurrency. After successful processing, call the booking service to create the booking and update the seat count in both the concert and seat type. If an error occurs, retry updating the seat count.
To keep seat data synchronized between Redis and MongoDB, data integrity should be checked periodically — for example, every 5 minutes

5. System must ensure:
    * No duplicate bookings per user per concert.
    * No overbooking, even with high concurrency.
# => I have used lua-script and leveraged the uniqueness of the model. Regarding the use of lua-script, it only works effectively if the system doesn't handle too many requests. I tested with Canon 22 requests per 1 second for 10 seconds, and 4 invalid requests slipped through. In this case, there are several ways to handle it. I could use RabbitMQ to ensure that only one request is processed at a time.

6.  Automatically disable bookings once the concert starts
 - Endpoint: /v1/concerts/inactive
 # - This cron job will run at the beginning of the day.

7. Cancel booking API (free up seats)
- POST /v1/seat-types/cancel-booking
# - Description: Similarly to booking creation, instead of decreasing the seat count, increase the seat count by 1 when canceling a booking.

8. Simulated email confirmation
- triggerSendMail on controllers/booking.controllers
# - Description: It will be called asynchronously and pushed into a queue.

9. canon test
  npm install -g autocannon
 autocannon -c 200 -d 5 -m POST http://api.minhnguyen.info.vn/v1/seat-types/booking \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjp7ImlkIjoiNjgxZjIyZDliN2RhNGE0NmI5OTVlYjE3IiwiZnVsbF9uYW1lIjoibWluaG5ndXllbiIsImVtYWlsIjoibWluaG4xbjFmNDIwM0BnbWFpbC5jb20iLCJwaG9uZSI6IjA5MTIzNzk4MTIiLCJsYXN0X2xvZ2luIjoxNzQ2ODcyNzU5MzYyLCJ1dWlkIjoiY2RkMDIyNGU1ZmQ4M2RmOTMxN2Y0NTY1NDU5NTllNTgifSwiaWF0IjoxNzQ2ODcyNzU5LCJleHAiOjE3NDc0Nzc1NTl9.iXuDuUKIj-XEABresGyULkb_XtKhJdyEvm3zbE-8Tvg" \
  -b '{"concert_id": "6820b33a9df9e87fa4d13897", "seat_type": "3"}' \
  --renderStatusCodes \
  --output results.txt

# POSTMAN
# flow you can replace the domain minhnguyen.info.vn with api.localhost if you're running locally. The APIs I handle are only designed for happy-case scenarios.

# register
curl --location 'http://api.minhnguyen.info.vn/v1/users/register' \
  --header 'Content-Type: application/json' \
  --request POST \
  --data-raw '{
    "phone": "0123890123",
    "email": "minh123@gmail.com",
    "password": "123123",
    "full_name": "minguyen"
  }'

# login
curl --location 'api.minhnguyen.info.vn/v1/users/login' \
--header 'Content-Type: application/json' \
--data '{
    "email_or_phone": "0123890123",
    "password": "123123"
}'

# create concert -> use Avian-Access-Trusted
curl --location 'api.minhnguyen.info.vn/v1/concerts' \
--header 'Content-Type: application/json' \
--header 'Avian-Access-Trusted: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCxRpoY9wwslWplvyV8B4bi+f+FSBEvPT3YlBc56d5EJK9i1Mq+Spcq3efMis/JqRvHIjGLErKgciYATKILqmFBhMX5Sri+KNjDk0saeqQTFVEUDKuNrcl18PVsCnvFQNLd49p7eqxH0uYgLdKFOLg/zOFmTUxL/v8WIHLPVsiyztYASNZZ38Yio/URvCQA9PSLeb3tHFwGpzr4YzjnB0GBymN5gJInn4YLpbVlNlqEcsn3SoJOgb8CFMT7SurN/7vFqLSQ5RO7qP7ayHtpKELojuWCBa3C8LIHGmeIAM6xuwdyGfJ0Pjc/4WRs2ZyTNKKytmGT185URR2R6es/eGd5XyZG0D1luPC3Z2H+ntwaJYBwPutWXKjoucL+fNS1jYruwBeMoMPzH1ic6YfLYG+NJ9+kRBSxWqNnVVw2IV/1fAs+T3jV0rptQiFd4Jm8mb9Db8HNFniHxcFnRO+dU0k9zKXd4FOs19wJVqVHaCvamsRamoWB/QwxqQ4z9FFj1Gut6U3Ks1+4ZojSCXjZ8ERlKFo85/8pjvncOxbvX+bUw6aRCBCntNyr4cPXxR0BqSTHkHRzt4apJNX1XpysvWn0HgT5D0fvyhi32JdZCMzo168uSj5L8dL2X/1NwmCqsSEI1nZMdUwVwTZ8KRe4xnOE9GOLlcg7IBuyiwrfyzRd2w==' \
--data '{
  "name": "SontungMPT World Tour",
  "description": "Live concert by Blackpink",
  "date": "2025-12-01T19:00:00.000Z",
  "location": 1,
  "artists": "SontungMTP",
  "seat_types": [
    {
      "type": 3,
      "name": "VIP Zone",
      "price": 250,
      "total_seats": 50,
      "remaining_seats": 50
    },
    {
      "type": 1,
      "name": "Regular Zone",
      "price": 150,
      "total_seats": 100,
      "remaining_seats": 100
    },
    {
      "type": 5,
      "name": "General Admission",
      "price": 100,
      "total_seats": 200,
      "remaining_seats": 200
    }
  ]
}
'
# get list concert
curl --location 'api.minhnguyen.info.vn/v1/concerts?populate_field=seat_types&status=1' \
--header 'Content-Type: application/json' \
--header 'Avian-Access-Trusted: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCxRpoY9wwslWplvyV8B4bi+f+FSBEvPT3YlBc56d5EJK9i1Mq+Spcq3efMis/JqRvHIjGLErKgciYATKILqmFBhMX5Sri+KNjDk0saeqQTFVEUDKuNrcl18PVsCnvFQNLd49p7eqxH0uYgLdKFOLg/zOFmTUxL/v8WIHLPVsiyztYASNZZ38Yio/URvCQA9PSLeb3tHFwGpzr4YzjnB0GBymN5gJInn4YLpbVlNlqEcsn3SoJOgb8CFMT7SurN/7vFqLSQ5RO7qP7ayHtpKELojuWCBa3C8LIHGmeIAM6xuwdyGfJ0Pjc/4WRs2ZyTNKKytmGT185URR2R6es/eGd5XyZG0D1luPC3Z2H+ntwaJYBwPutWXKjoucL+fNS1jYruwBeMoMPzH1ic6YfLYG+NJ9+kRBSxWqNnVVw2IV/1fAs+T3jV0rptQiFd4Jm8mb9Db8HNFniHxcFnRO+dU0k9zKXd4FOs19wJVqVHaCvamsRamoWB/QwxqQ4z9FFj1Gut6U3Ks1+4ZojSCXjZ8ERlKFo85/8pjvncOxbvX+bUw6aRCBCntNyr4cPXxR0BqSTHkHRzt4apJNX1XpysvWn0HgT5D0fvyhi32JdZCMzo168uSj5L8dL2X/1NwmCqsSEI1nZMdUwVwTZ8KRe4xnOE9GOLlcg7IBuyiwrfyzRd2w=='

# get detail concert
curl --location 'api.minhnguyen.info.vn/v1/seat-types?concert_id=6821e93f457d2998c4a384e4' \
--header 'Content-Type: application/json' \
--header 'Avian-Access-Trusted: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCxRpoY9wwslWplvyV8B4bi+f+FSBEvPT3YlBc56d5EJK9i1Mq+Spcq3efMis/JqRvHIjGLErKgciYATKILqmFBhMX5Sri+KNjDk0saeqQTFVEUDKuNrcl18PVsCnvFQNLd49p7eqxH0uYgLdKFOLg/zOFmTUxL/v8WIHLPVsiyztYASNZZ38Yio/URvCQA9PSLeb3tHFwGpzr4YzjnB0GBymN5gJInn4YLpbVlNlqEcsn3SoJOgb8CFMT7SurN/7vFqLSQ5RO7qP7ayHtpKELojuWCBa3C8LIHGmeIAM6xuwdyGfJ0Pjc/4WRs2ZyTNKKytmGT185URR2R6es/eGd5XyZG0D1luPC3Z2H+ntwaJYBwPutWXKjoucL+fNS1jYruwBeMoMPzH1ic6YfLYG+NJ9+kRBSxWqNnVVw2IV/1fAs+T3jV0rptQiFd4Jm8mb9Db8HNFniHxcFnRO+dU0k9zKXd4FOs19wJVqVHaCvamsRamoWB/QwxqQ4z9FFj1Gut6U3Ks1+4ZojSCXjZ8ERlKFo85/8pjvncOxbvX+bUw6aRCBCntNyr4cPXxR0BqSTHkHRzt4apJNX1XpysvWn0HgT5D0fvyhi32JdZCMzo168uSj5L8dL2X/1NwmCqsSEI1nZMdUwVwTZ8KRe4xnOE9GOLlcg7IBuyiwrfyzRd2w=='

# create booking 
curl --location 'api.minhnguyen.info.vn/v1/seat-types/booking' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjp7ImlkIjoiNjgxZjIyZDliN2RhNGE0NmI5OTVlYjE3IiwiZnVsbF9uYW1lIjoibWluaG5ndXllbiIsImVtYWlsIjoibWluaG4xbjFmNDIwM0BnbWFpbC5jb20iLCJwaG9uZSI6IjA5MTIzNzk4MTIiLCJsYXN0X2xvZ2luIjoxNzQ2ODcyNzU5MzYyLCJ1dWlkIjoiY2RkMDIyNGU1ZmQ4M2RmOTMxN2Y0NTY1NDU5NTllNTgifSwiaWF0IjoxNzQ2ODcyNzU5LCJleHAiOjE3NDc0Nzc1NTl9.iXuDuUKIj-XEABresGyULkb_XtKhJdyEvm3zbE-8Tvg' \
--data '{
    "concert_id": "6821e93f457d2998c4a384e4",
    "seat_type": "3"
}'

# cancel booking
curl --location 'api.minhnguyen.info.vn/v1/seat-types/cancel-booking' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjp7ImlkIjoiNjgxZjIyZDliN2RhNGE0NmI5OTVlYjE3IiwiZnVsbF9uYW1lIjoibWluaG5ndXllbiIsImVtYWlsIjoibWluaG4xbjFmNDIwM0BnbWFpbC5jb20iLCJwaG9uZSI6IjA5MTIzNzk4MTIiLCJsYXN0X2xvZ2luIjoxNzQ2ODcyNzU5MzYyLCJ1dWlkIjoiY2RkMDIyNGU1ZmQ4M2RmOTMxN2Y0NTY1NDU5NTllNTgifSwiaWF0IjoxNzQ2ODcyNzU5LCJleHAiOjE3NDc0Nzc1NTl9.iXuDuUKIj-XEABresGyULkb_XtKhJdyEvm3zbE-8Tvg' \
--data '{
    "concert_id": "6821e93f457d2998c4a384e4"
}'


# another api
curl --location 'api.minhnguyen.info.vn/v1/booking' \
--header 'Content-Type: application/json' \
--header 'Avian-Access-Trusted: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCxRpoY9wwslWplvyV8B4bi+f+FSBEvPT3YlBc56d5EJK9i1Mq+Spcq3efMis/JqRvHIjGLErKgciYATKILqmFBhMX5Sri+KNjDk0saeqQTFVEUDKuNrcl18PVsCnvFQNLd49p7eqxH0uYgLdKFOLg/zOFmTUxL/v8WIHLPVsiyztYASNZZ38Yio/URvCQA9PSLeb3tHFwGpzr4YzjnB0GBymN5gJInn4YLpbVlNlqEcsn3SoJOgb8CFMT7SurN/7vFqLSQ5RO7qP7ayHtpKELojuWCBa3C8LIHGmeIAM6xuwdyGfJ0Pjc/4WRs2ZyTNKKytmGT185URR2R6es/eGd5XyZG0D1luPC3Z2H+ntwaJYBwPutWXKjoucL+fNS1jYruwBeMoMPzH1ic6YfLYG+NJ9+kRBSxWqNnVVw2IV/1fAs+T3jV0rptQiFd4Jm8mb9Db8HNFniHxcFnRO+dU0k9zKXd4FOs19wJVqVHaCvamsRamoWB/QwxqQ4z9FFj1Gut6U3Ks1+4ZojSCXjZ8ERlKFo85/8pjvncOxbvX+bUw6aRCBCntNyr4cPXxR0BqSTHkHRzt4apJNX1XpysvWn0HgT5D0fvyhi32JdZCMzo168uSj5L8dL2X/1NwmCqsSEI1nZMdUwVwTZ8KRe4xnOE9GOLlcg7IBuyiwrfyzRd2w==' \
--data '{
    "seat_type_id": "6820a7a61fae79fe5d60f378",
    "concert_id": "6820a7a61fae79fe5d60f377",
    "concert_name": "Blackpink World Tour",
    "price": 250,
    "seat_name": "VIP Zone",
    "total_seats": 50,
    "remaining_seats": 50,
    "created_at": "2025-05-11T13:35:34.937Z",
    "modified_at": "2025-05-11T13:35:34.937Z",
    "__v": 0,
    "user_id": "681f22d9b7da4a46b995eb17"
}'

curl --location 'api.minhnguyen.info.vn/v1/booking' \
--header 'Content-Type: application/json' \
--header 'Avian-Access-Trusted: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCxRpoY9wwslWplvyV8B4bi+f+FSBEvPT3YlBc56d5EJK9i1Mq+Spcq3efMis/JqRvHIjGLErKgciYATKILqmFBhMX5Sri+KNjDk0saeqQTFVEUDKuNrcl18PVsCnvFQNLd49p7eqxH0uYgLdKFOLg/zOFmTUxL/v8WIHLPVsiyztYASNZZ38Yio/URvCQA9PSLeb3tHFwGpzr4YzjnB0GBymN5gJInn4YLpbVlNlqEcsn3SoJOgb8CFMT7SurN/7vFqLSQ5RO7qP7ayHtpKELojuWCBa3C8LIHGmeIAM6xuwdyGfJ0Pjc/4WRs2ZyTNKKytmGT185URR2R6es/eGd5XyZG0D1luPC3Z2H+ntwaJYBwPutWXKjoucL+fNS1jYruwBeMoMPzH1ic6YfLYG+NJ9+kRBSxWqNnVVw2IV/1fAs+T3jV0rptQiFd4Jm8mb9Db8HNFniHxcFnRO+dU0k9zKXd4FOs19wJVqVHaCvamsRamoWB/QwxqQ4z9FFj1Gut6U3Ks1+4ZojSCXjZ8ERlKFo85/8pjvncOxbvX+bUw6aRCBCntNyr4cPXxR0BqSTHkHRzt4apJNX1XpysvWn0HgT5D0fvyhi32JdZCMzo168uSj5L8dL2X/1NwmCqsSEI1nZMdUwVwTZ8KRe4xnOE9GOLlcg7IBuyiwrfyzRd2w==' \
--data '{
    "concert_id": "6820b33a9df9e87fa4d13897",
    "user_id": "681f22d9b7da4a46b995eb17"
}'
