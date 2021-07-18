# JavaScript Live Project

## Introduction

For the last two weeks of my time on the JavaScript (including an advanced JavaScript course), I worked in an Agile/Scrum environment to build a tourism website to highlight activities for a particular city. We used Azure DevOps to manage user stories and code repos, and related knowledge about the project. The website was built mainly with Angular 12 and Bootstrap 5 on the front-end, and Node.JS, Express.JS, Mongoose, and MongoDB (Atlas) on the back-end.

I've worked with HTML and CSS previously, but not at this level. So building the front-end from scratch was new for me and provided a great learning experience with Angular, CSS, Bootstrap, and TypeScript. I built several components to display tiles for each restaurant and tour in a basic responsive fashion, and a data service to retrieve the different sets of data.

I was able to leverage a lot of my previous work experience with Node.JS and Express.JS to quickly build the back-end RESTful API/controllers to service the website. I learned more about Mongoose to build the model to store and retrieve information about restaurants and tours available for the city, such as a thumbnail image, short description, and a link to the related website. I also added the current temperature and weather condition icon to the navbar for which I built a controller to pull the weather data from an external REST API.

Below are some of the highlights of the project.

## Back-end
- [Weather Condition Controller](Weather-Condition-Controller)
- [Restaurant Controller](Restaurant-Controller)

## Weather Condition Controller
This controller was designed to simply pull the current weather icon and temperature from an external REST API to be displayed on the front-end navbar.

```javascript
const request = require('request');

exports.getLocalWeather = (req, routeRes) => {
    request(process.env.WEATHER_API,
        { json: true },
        (err, res, body) => {
            if (err) { 
                return routeRes.status(500).send({ message: 'Unable to fetch weather data' });
            }
            console.log(body);
            return routeRes.status(200).send({
                current: {
                    temp_f: body.current.temp_f,
                    icon: body.current.condition.icon
                }
            });
        });
};
```

## Restaurant Controller
The restaurant controller utilized a Mongoose data model and retrieved/stored json data about individual restaurants.
```javascript
const Restaurant = require('../models/restaurantModel');

exports.create = (req, res) => {
    // Validate request
    if (!req.body.name || !req.body.websiteUrl || !req.body.description || !req.body.imageUrl) {
        return res.status(400).send({
            message: 'Required fields can not be empty',
        });
    }

    // Create new restaurant object from request body
    const newRestaurant = new Restaurant({
        name: req.body.name,
        websiteUrl: req.body.websiteUrl,
        description: req.body.description,
        imageUrl: req.body.imageUrl
    });
  
    // Save new restaurant data to db
    newRestaurant.save()
        .then((restaurant) => {
            // TODO: Should new record be returned?
            return res.status(201).send({
                restaurant: restaurant
            });
        })
        .catch((err) => {
            // TODO: Move to error handler and refine error handling for specific errors
            res.status(500).send({
                message: err.message || 'An unknown error occurred saving new restaurant data'
            });
        });
};

exports.findAll = (req, res) => {
    Restaurant.find()
        .sort({ name: 1 })
        .then((restaurants) => {
            res.status(200).send(restaurants);
        })
        .catch((err) => {
            // TODO: Move to error handler and refine error handling for specific errors
            res.status(500).send({
                message: err.message || 'An unknown error occurred retrieving restaurant data'
            });
        });
};

exports.deleteById = (req, res) => {
    Restaurant.findByIdAndRemove(req.params.id)
        // TODO: Move to error handler and refine error handling for specific errors
        .then((restaurant) => {
            if (!restaurant) {
                return res.status(404).send({
                    message: `Restaurant with id: ${req.params.id} not found`
                });
            }
            return res.status(204).send();
        })
        .catch((err) => {
            return res.status(500).send({
                message: `Failed to update restaurant with id: ${req.params.id}, error: ${err.message}`
            });
        });
};


exports.updateById = (req, res) => {
    if (!req.body.name && !req.body.websiteUrl && !req.body.description && !req.body.imageUrl) {
        return res.status(400).send({ message: 'Required fields cannot be empty' });
    }
    Restaurant.findByIdAndUpdate(req.params.id, req.body, { new: true })
        .then((restaurant) => {
            if (!restaurant) {
                return res.status(404).send({
                    message: `Restaurant with id: ${req.params.id} not found`
                });
            }            
            return res.status(200).send({
                restaurant: restaurant
            });
        })
        .catch((err) => {
            // TODO: Move to error handler and refine error handling for specific errors
            return res.status(500).send({
                message: `Failed to update restaurant with id: ${req.params.id}, error: ${err.message}`
            });
        });
};
```

## Front-end
- [Tile Component](tile-component)
- [Data service](data-service)

## Tile Component
I built a component that would render a tile with an animated thumbnail image, short description and an external link (the restaurant or tour in this case), so that changes to the design could be easily made anywhere it was used.
```javascript
import { Component, Input, OnInit } from '@angular/core';

@Component({
  selector: 'app-tile',
  templateUrl: './tile.component.html',
  styleUrls: ['./tile.component.css'],
})
export class TileComponent implements OnInit {
  @Input() imageUrl!: string;
  @Input() cardTitle!: string;
  @Input() cardText!: string;
  @Input() websiteUrl!: string;

  constructor() {}

  ngOnInit(): void {

  }
}

```
```html
    <div class="card h-100" 
        placement="top"
        ngbPopover={{cardText}} 
        popoverTitle={{cardTitle}}
    >
      <img [src]="imageUrl" class="card-img-top">
      <div class="card-body">
        <h5 class="card-title fw-bolder">{{cardTitle}}</h5>
        <p class="card-text text-start scroll">{{cardText}}</p>
      </div>
      <div class="card-footer">
        <small class="text-muted">
          <a [href]="websiteUrl">
            <fa-icon [icon]="['fas', 'external-link-alt']"></fa-icon>&nbsp;
            Website
          </a>
        </small>
      </div>
    </div>
```

## Data Service
A data service was constructed to handle retrieving various data from the backend, such as image path, a short description and link for the tiles, and the weather data.
```typescript
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable, throwError } from 'rxjs';
import { catchError, map, tap } from 'rxjs/operators';
import { AppConfigService } from './app-config.service';
@Injectable({
  providedIn: 'root',
})
export class DataService {
  carouselImagesApiUrl: string;
  restaurantsApiUrl: string;
  toursApiUrl: string;
  bostonWeatherApiUrl: string;

  constructor(private http: HttpClient, appConfig: AppConfigService) {
    // Load app config data
    this.carouselImagesApiUrl =
      appConfig.apiBaseUrl + appConfig.carouselImagesApiUrl;
    this.restaurantsApiUrl =
      appConfig.apiBaseUrl + appConfig.restaurantsApiUrl;
    this.toursApiUrl =
      appConfig.apiBaseUrl + appConfig.toursApiUrl;
    this.bostonWeatherApiUrl = appConfig.weatherApiUrl;

  }

  getCarouselImages(): Observable<any[]> {
    return this.http
      .get<any[]>(this.carouselImagesApiUrl)
      .pipe(tap((data) => {}, catchError(this.handleError)));
  }

  getRestaurants(): Observable<any[]> {
    return this.http
      .get<any[]>(this.restaurantsApiUrl)
      .pipe(tap((data) => {}, catchError(this.handleError)));
  }

  getTours(): Observable<any[]> {
    return this.http
      .get<any[]>(this.toursApiUrl)
      .pipe(tap((data) => {}, catchError(this.handleError)));
  }

  getLocalWeather(): Observable<any[]> {
    return this.http
      .get<any[]>(this.bostonWeatherApiUrl)
      .pipe(tap((data) => {}, catchError(this.handleError)));
  }

  private handleError(err: HttpErrorResponse): Observable<never> {
    // TODO: At some point this should be sent to a logging service instead
    // of just logging it to the console.
    let errorMessage = '';
    if (err.error instanceof ErrorEvent) {
      // A client-side or network error occurred. Handle it accordingly.
      errorMessage = `An error occurred: ${err.error.message}`;
    } else {
      // The backend returned an unsuccessful response code.
      // The response body may contain clues as to what went wrong,
      errorMessage = `Server returned code: ${err.status}, error message is: ${err.message}`;
    }
    console.error(errorMessage);
    return throwError(errorMessage);
  }
}
```

## Summary
This was a short but productive 2-week sprint, and I really enjoyed my time delving into the front-end frameworks of Angular 12 (building components, working with routing and nested routing, TypeScript, and Observables), Bootstrap 5 (continually coercing it and CSS to do what I wanted it to), and Fontawesome for some addition icons.

As always, I really enjoyed standing up a back-end RESTful API with Node.js and Express.js. One of my favorite things to do is connecting systems together (see my [Conductor 2](https://github.com/aihrig/Conductor2) project connecting a LEGO train via pub/sub to a front-end). This project gave me the opportunity become more familiar with front-end frameworks and ensuring that both the front and back-end systems worked together.

Jump to [Top](Live-Project), [Back-end](back-end), [Front-End](front-end)