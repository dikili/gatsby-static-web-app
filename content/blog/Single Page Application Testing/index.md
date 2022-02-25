---
title: Single Page Application Testing
date: 2022-02-27 19:40:00 +0300
description: # Add post description (optional)
img: ./SinglePage.PNG # Add image post (optional)
tags: [] # add tag
---


SPA stands for single page application, in this article we will focus on the UI SPA testing, thus I assume you already know what are SPAs , how they work. 

With testing the SPAs, it is not uncommon that some component of the SPA is neglected to be tested throughly which easily 'should' have been an easy gain turns into a easy loss (yes exactly that bug went to live I am talking about :) ) at the blink of an eye. Reasons are often not quiet understanding all the moving pieces of a SPA as well, hence lets look at the various tests that we need to ensure we have (and communicate this to the person that does e2e or integration tests so no duplications) that are addressed to different parts of the SPA.

Tests
--------

### Unit Tests

Well when we say SPA Unit tests, although this is often mixed and mistaken to be component tests itself, in actuality this is a different.
Unit tests in an SPA is a test that we put to test our classes or functionality for a single unit (class). This is not related to the component(s) at all.
take an example below for a clearer explanation. Lets say we have an Event helper class with a 'Add' function on it, which supposed to add an event, if we were to test this, which we should it should start with the unit test itself.
e.g
```
export interface AddEventResult {
    eventAdded: boolean,
}

class Event {

    private _name: string;
    private _location: string;
    private _added: boolean;
    private _expired: boolean;

    constructor(location: string, expired: boolean = false) {
        this._location = location;
        this._expired = expired;
    }

 
    add(events: Event[]): AddEventResult {
        this._added = true;
      
        return {
            eventAdded: this._added,
        }
    }
}

export default Event;

```

If you look at the class above, it does Add() Events, hence we wont know until we use this class, whether it adds events or not!! So we need to verify/validate and unit test this in reality to not face with any suprises that this class fails us. Lets tests this class then, since it is attempting to add some list of events and return whether it added or not lets come up with some test scenerios;

1) add some unexpired events;

```
it('should add some unexpired events', () => {
    const expired = false;
    const location = 'London';
    const event = new Event(location, expired);
    const { added } = event.Add([{'manchester'},{'brussels'}]);
    expect(added).toBeTruthy();
})
```

So , this unit test ensure that our add function works as expected and able to handle operations in isolation.

2) add some expired event
```
it('should add an unexpired event', () => {
    const expired = false;
    const location = 'London';
    const event = new Event(location, expired);
    const { added } = event.Add([{'london'},{'lisbon',true}]);
    expect(!added).toBeTruthy();
})
```

As you can see on the above test, we managed to find an issue with our class,  expect(!added).toBeTruthy(); has now failed, because our business logic wants us to ensure expire events can not be added and the second event ``` {'lisbon',true}``` our class has added that although it expired thus this has proven to be a very useful unit test.

sky is the limit for unit testing as in other types of testing but having reasonable amount of with the most ROI is what makes sense for unit testing SPAs.Few more test scenerios can be found below but of course there can be more.

--------
3) add an event without any location
4) add events for cities in africa 


### Single Component Tests

UI Components are the minumum building blocks of SPA(s). They are like the small tents in a camping site, each doing something different and passes that information to its neighbouring other components( and receiving or both) to achieve the business goal of the SPA. For react apps, React Testing Library or Enzyme will be needed to write component tests. I will follow from the Event example above.

Lets imagine the following event component that we created, On init, it gets all the events and renders them one at a time on the UI. 

```
@Component({

  selector: 'app-Event',
  templateUrl: './Event.component.html',
  styleUrls: ['./Event.component.css']
})
export class EventComponent implements OnInit {
  events: Event[];


  constructor(private eventService: EventsService,
    private alertifyService: AlertifyService) { }

  ngOnInit() {
    this.getEvents();
  }

  getEvents() {
    this.eventService
    .getEvents()
     .subscribe((events) => {
       console.log('getting events');
       console.log(events);
       this.events = events;
     }, error => {
       console.log('api error');
       this.alertifyService.error(error);
     });
  }

}

```

As you can see above this component is responsible from getting list of events and rendering this on the UI. So we should write a component test to verify that component is able to fetch events should be the test here (each component is different but whatever that component is doing we should check if it is able to do so or not), we should also have a test if the component can be created or not as below;

```
describe('EventComponent', () => {
  let component: EventComponent;
  let fixture: ComponentFixture<EventComponent>;

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      declarations: [ EventComponent ]
    })
    .compileComponents();
  }));

  beforeEach(() => {
    fixture = TestBed.createComponent(EventComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    component.getEvents()
    expect(component).toBeTruthy();
  });

  it('able to fetch events', () => {
    const events = component.getEvents();
    expect(events).toBeTruthy();
  });

});
```

Now we also learned what a single component test is and does. Again, there can be component level other tests as well e.g has it mounted etc.. thus component specific tests can go into this category.


### Component Integration Tests

Component Integration tests are also component related tests but as the name suggests we are not testing a single isolated component solely this time but we are testing the integration between different components. After we test each component in isolation it is time to check if components are being used properly in the context of another component.Child component, parent component relation or the points where component subscribes to another in a way for data retriaval and processing, hence these tests will focus on these integration points and ensure that correct data passing would yield in accurate behaviour and rendering in components.

If we want to sample this out; Event component has a EventDetail child component as below, which it passes event information to.

```
export class EventComponent implements OnInit {
  @ViewChild('eventDetails') eventDetails: TabsetComponent;
  event: Event;
    galleryOptions: NgxGalleryOptions[];
    galleryImages: NgxGalleryImage[];

  constructor(private userService: UserService, private alertifier: AlertifyService, private router: ActivatedRoute) { }

  ngOnInit() {
   this.router.data.subscribe(data => {
     this.event = data['event'];
   });

   this.router.queryParams.subscribe(params => {
     const selectedTab = params['tab'];
     this.eventDetails.tabs[selectedTab > 0 ? selectedTab : 0].active = true;
   });

  this.galleryOptions = [
    {
        width: '500px',
        height: '500px',
        thumbnailsColumns: 4,
        imagePercent: 100,
        imageAnimation: NgxGalleryAnimation.Slide,
        preview: false
    },
    ];

    this.galleryImages = this.getImages();

  }
  getImages() {
    const images = [];
    for (let i = 0; i < this.user.photos.length; i++) {
           images.push({
             small: this.event.photos[i].url,
             medium : this.event.photos[i].url,
             large: this.event.photos[i].url,
             description: this.event.photos[i].description
           });
    }

    return images;
   }

   selectTab(tabId: number) {
     this.eventDetails.tabs[tabId].active = true;
   }

}
```
Now we can have a component integration test as in below that for the select event, corresponding event details are rendered.

```
it('should view event details', () => {
    const event = new Event([{'london'}]);

    const { container } = render(<EventComponent event={event} />)

    const eventDetail = container.querySelectorAll('.tab')[0];
    fireEvent.click(eventDetail);

    expect(eventDetail.tabs[0].active).toBe(true);
})
```

Conclusion
----------

We went over 3 types of tests here, SPA obviously have BFF part as well which we wont cover in this article but for the UI part of the SPA , we have covered all aspects of a complete testing suite for SPAs. Types of testing each play their unique role in achieving bug-free releases.