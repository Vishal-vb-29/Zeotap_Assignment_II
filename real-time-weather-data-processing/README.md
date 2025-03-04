# Application 2 : Real-Time Data Processing System for Weather Monitoring with Rollups and Aggregates
> Zeotap | Software Engineer Intern | Assignment | Application 2

## Applicant Introduction
Hi! I am Vishal, I want to point to my 
[LinkedIn](https://www.linkedin.com/in/vishal-b-4606a7135/).



## Introduction
> I read the assignment description multiple times to make sure I didn’t miss anything important. 


Most of my code is written in a detailed way, so I kept the README straightforward and in bullet points:

The code is heavily commented and includes clear docstrings (Python with Sphinx).
Unittests cover both positive and negative edge cases, with test coverage kept above 80%.
This README starts with technical details, followed by the solution, and finally covers other relevant points, which, while technical, are in the “non-technical” section. This part shares my thought process and approach, along with a brief design discussion.

## Technical Parts
### Installation
I have tried my best to make it platform agnostic, and packaged everything into a 
Docker Container. You can also execute the below seperately for running them on 
your machine without Docker.

+ Backend

    If `poetry` isn't previously installed, install it first. 
    ```bash
    python3 -m venv $VENV_PATH
    $VENV_PATH/bin/pip install -U pip setuptools
    $VENV_PATH/bin/pip install poetry
    ```

    Then continue to create a venv, and install dependencies
    ```bash
    cd weather-service/
    poetry install
    ```

+ Database

    After a `poetry install`, execute the following to setup raw data in your 
    postgres instance. Make sure to populate the connection creds in the `.env`.
    Run the database first, if using manual methods
    ```bash
    docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres
    ```

### How to run
Expecting that above installation process, suceeded.

+ Backend
    ```bash
    poetry run python main.py --dev
    ```
+ Database
    ```bash
    docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres
    ```

or you can just run the `Dockerfile` to build the system and run it in a container

```bash
cd weather-service/
docker build -t weather_service_image .
docker run --name weather_service_container -d -p 8000:8000 weather_service_image

# Check the logs to ensure everything is running smoothly
docker logs -f weather_service_container
```

## About Solution
The problem wanted us to create a service thats forever running and updates weather data every 5 
minutes. The most intuitive and simple way of doing it is long polling, and that is what the problem
overview also suggests, poll every 5 mins for infinite time. Now, there are many disadvantages to this 
method, but for the given use case, it makes most sense. The following could be the challenges - 
* Failover/Resiliency - what if the service crashes in the middle, how would we failover to other 
  instances. One simple way we can acheive this is multiple "cheap" instances of the service, 
  sort of a pool of worker, where one is master. The master serves write loads and reads are spread
  across instances. Now, we start a simple heartbeat service, which pings the master every few minutes 
  (<5 mins of course,) if the request fails, one from the pool is made a master. For this we should 
  have atleast N+2 = 3 instances (N+1 failover + 1 which takes in updates on the go.)
+ High Read loads - what if the read load is high? we can put a cache in front of the database, that would
  server the reads. Now, the TTL policy could be 5 minutes, because every 5 minutes the data is going to 
  change, but for the entirety of the 5 mins, its going to be constant. 
+ What about write loads? Write load is going to be constant, because we'll be writing only once in 5 mins.
+ Alert mechanism - Right now the alert mechanism only return a list in a JSON format, a UI based (using 
  Plotly), and an HTML Table format. So users can simply connect to the JSON endpoint and do long polling
  if they want to consume it via any client application. SSEs could be a better option, but I didn't have 
  time to implement that. 
  As far as the architecture for sending mails is concerned, we can have a worker which continously read 
  this endpoint, and dump every alert into a Kafka Queue. On the other side, we can have a pool of workers
  which consume these one by one, and mail it to the user. This would be a real-time/"online" solution.
  But the other way could be, since we its going to be only once every 5 minutes, we'll wait for the duration
  and batch these alerts together. Batch processing them would be efficient.

### Solution Overview
To solve the problem, I created a simple API service. 
* I used FastAPI (Python) to create the API, it uses an ASGI Server to handle requests.
* It uses an Async Job Scheduler to schedule daily aggregation cron job (at midnight 12:01) and long polling
  job (at an interval of 300 secs or 5 mins.)
* After each long poll, the data is ingested into a `RealtimeWeather` table. This data is checked against
  alert thresholds (`Thresholds` class), and then if any of the thresholds is crossed, it updates in the 
  `AlertEvents` Table. 
* The aggregation queries data from `RealtimeWeather` table, aggregates and dumps to `DailyWeather` table.
* Now during each ingestion of `RealtimeWeather`, old records beyond 24 hours are deleted. It only contains
  latest 24 hours of data, on a rolling basis.
* All the visualizations are served over a WSGIMiddleware using `dash` and `plotly`. They are clean, and 
  interactive with less code size. 
* We use SQLAlechemy as the ORM, so all transactions made via API, are almost always consistent.

### Code Structure
file_path | file_description
----------|-------------------
__init__.py|for top-level module declaration
Dockerfile | packaging as a Docker container
main.py | run this to execute commands on the application, eg running server, tests
poetry.lock | necessary for installing packages via poetry
pyproject.toml| project metadata, also needed for poetry
weather_service/dash_app_alerts.py| Dash App to render alerts in a table.
weather_service/dash_app_statistics.py| Dash App to render charts (historical and realtime)
weather_service/dash_app_threshold.py | Table to render and change thresholds
weather_service/db_models.py | Contains Table, and Database schema
weather_service/db_utils.py | Utility Functions to read/write data into Database
weather_service/main.py | API Contracts, and everything to run the service
weather_service/utils.py | Utility functions used in API, eg checking against thresholds, etc

#### Database Design
Tables
+ `realtime_weather`
  Columns:
    - dt (TIMESTAMP): Date and time of the weather report.
    - main_condition (String): Main weather condition description.
    - temp (Numeric): Temperature in degrees.
    - feels_like (Numeric): Perceived temperature.
    - pressure (Numeric): Atmospheric pressure.
    - humidity (Numeric): Humidity percentage.
    - rain (Numeric): Rainfall amount.
    - clouds (Numeric): Cloudiness percentage.
    - city (String): City name.
    - PRIMARY KEY (dt, city)

+ `daily_weather`
  Columns:
    - date (Date): The date of the weather report.
    - city (String): City name.
    - avg_temp (Numeric): Average temperature for the day.
    - max_temp (Numeric): Maximum temperature for the day.
    - min_temp (Numeric): Minimum temperature for the day.
    - dom_condition (String): Dominant weather condition for the day.
    - PRIMARY KEY (date, city)

+ `alert_events`
  Columns:
    - event_id (UUID): Unique identifier for the alert event. PRIMARY KEY
    - dt (TIMESTAMP): Date and time of the alert.
    - city (String): City name where the alert was issued.
    - reason (String): Reason for the alert.
    - trigger (String): The condition that triggered the alert.

## API Design
* GET / - Health check endpoint.
* GET /alerts/ - Retrieve alerts in a UI friendly format.
* GET /alerts/json/ - Retrieve alerts in JSON format. 
* GET /alerts/html/ - Retrieve alerts in HTML format.
* GET /statistics - Visualize Historical and Realtime weather data
* GET /configs/ - Retrieve and Update Thresholds Configurations

## Non Technical Parts
Most points are covered in the technical section, but this part explains my approach to solving the problem

### Approach

"When I first saw the problem, my initial thought was about the system’s overall design. But I realized it was more about implementing the APIs to handle requests. I would still outline my ideal design for this system if it were an interview, although I didn’t have the time to fully implement it."



### Feedback
The assignment was quite long and had several tricky parts that were harder to implement than expected. Due to limited time, I couldn’t fully cover availability guarantees, redundancy, or efficient caching for reads.

Thanks.
