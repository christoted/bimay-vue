<template>
    <div class="home lg:h-screen pt-16">
        <div class="grid grid-cols-12 h-full lg:overflow-y-hidden">

            <perfect-scrollbar :options="{handlers: ['click-rail', 'drag-thumb', 'keyboard', 'wheel']}" class="pt-4 lg:overflow-y-scroll h-full xl:col-span-9 lg:col-span-8 col-span-12 w-full px-6 md:px-8 lg:px-10">
                <div class="card w-full p-6 mb-12">
                    <div class="flex flex-col-reverse sm:flex-row justify-between">
                        <div class="flex flex-col justify-center w-full md:w-8/12">
                            <h3>{{greetings}}, <span class="capitalize">{{ binusianData.FIRST_NAME | lowerCase }}</span>!</h3>
                            <p class="pt-2">{{randomQuote}}</p>
                            <div v-if="nextClass">
                                <h4 class="mt-4">Your next class starting {{ nextClass.startDate | relativeTime }}.</h4>
                                <div class="text-gray-600 text-sm">
                                    <div><i class="mr-1 fas fa-book"/> {{nextClass.COURSE_TITLE_LONG}}</div>
                                    <div><i class="mr-1 fas fa-clock"/> {{nextClass.MEETING_TIME_START}} - {{nextClass.MEETING_TIME_END}}</div>
                                </div>
                            </div>
                            <div v-if="!nextClass">
                                <h4 class="mt-4">Time to relax. There's no class today.</h4>
                            </div>
                        </div>
                        <div class="flex mb-4 sm:mb-0 justify-center">
                            <img class="lg:max-h-full h-64 mr-0 object-contain" src="../../assets/img/study.png" alt="study"/>
                        </div>
                    </div>
                </div>

                <div class="mb-4">
                    <vue-content-loading v-if="videoConferences.length <= 0"  class="w-full" :height="60">
                        <rect x="0" y="0" width="50" height="12" />
                        <rect x="0" y="15" width="100" height="35" />
                        <rect x="105" y="15" width="100" height="35" />
                        <rect x="210" y="15" width="100" height="35" />
                    </vue-content-loading>

                    <video-conferences :video-conferences="videoConferences" :courses="courses" v-if="videoConferences.length > 0"/>
                </div>

                <div class="mb-4">
                    <vue-content-loading v-if="assignments.length <= 0"  class="w-full" :height="60">
                        <rect x="0" y="0" width="50" height="12" />
                        <rect x="0" y="15" width="100" height="35" />
                        <rect x="105" y="15" width="100" height="35" />
                        <rect x="210" y="15" width="100" height="35" />
                    </vue-content-loading>

                    <assignments :assignments="assignments" :courses="courses" v-if="assignments.length > 0"/>
                </div>
            </perfect-scrollbar>

            <perfect-scrollbar :options="{handlers: ['click-rail', 'drag-thumb', 'keyboard', 'wheel']}" class="pt-4 lg:overflow-y-scroll h-full pb-8 pl-6 lg:pl-2 xl:col-span-3 lg:col-span-4 col-span-12 pr-6 md:pr-8 lg:pr-4 xl:pr-10">
                <your-schedule :calendar-dates="calendarDates"/>
            </perfect-scrollbar>
        </div>
    </div>
</template>

<style scoped>

</style>

<script>
    // @ is an alias to /src
    // GET https://binusmaya.binus.ac.id/services/ci/index.php/staff/init/check_session
    // GET https://binusmaya.binus.ac.id/services/ci/index.php/general/getBinusianData
    // GET https://binusmaya.binus.ac.id/services/ci/index.php/student/init/getCourses
    // GET https://binusmaya.binus.ac.id/services/ci/index.php/student/classes/assignmentType/COMP6229/011643/1920/LEC/14910/01

    import axios from "axios";
    import * as HtmlTableToJson from "html-table-to-json";
    import moment from "moment-timezone";
    import Assignments from "../../components/Assignments";
    import { VueContentLoading } from 'vue-content-loading';
    import VideoConferences from "../../components/VideoConferences";
    import YourSchedule from "../../components/YourSchedule";
    import Repository from "../../repositories/RepositoryFactory";



    export default {
        name: 'Home',
        components: {YourSchedule, VideoConferences, Assignments, VueContentLoading},
        data() {
            return {
                binusianData: {},
                courses: [],
                videoConferences: [],
                classSchedules: [],
                assignments: [],
                isLoading: false,
                randomQuote: "",
                calendarDates: [
                    {
                        key: 'today',
                        highlight: true,
                        dates: new moment().toDate()
                    }
                ],
                nextClass: ""
            }
        },
        methods: {
            async checkSession() {
                const SessionFactory = Repository.get("session");

                this.$Progress.start();

                if (!this.$store.getters.isAuthenticated || this.$store.state.phpsessid === '') {
                    // redirect to login
                    await this.$store.dispatch('isAuthenticated', false);
                    await this.$router.push('/login');
                }

                const { data } = await SessionFactory.check();

                if (data.RoleID === 0) {
                    // redirect to login
                    await this.$store.dispatch('isAuthenticated', false);
                    await this.$router.push('/login');
                }

                this.$Progress.finish();
            },
            getBinusianData() {
                // https://binusmaya.binus.ac.id/services/ci/index.php/general/getBinusianData
                this.$Progress.start();

                axios.request({
                    url: "https://bimayproxy.herokuapp.com/fetch/https://binusmaya.binus.ac.id/services/ci/index.php/general/getBinusianData",
                    method: "get",
                    headers: {
                        Bisquit: `PHPSESSID=${this.$store.state.phpsessid}`
                    }
                })
                    .then((response) => {
                        this.binusianData = response.data;
                        this.$Progress.finish();
                    })
                    .catch((error) => {
                        console.log(error);
                        this.$Progress.fail();
                    });
            },
            getCourses() {
                this.$Progress.start();
                // https://binusmaya.binus.ac.id/services/ci/index.php/student/init/getCourses
                axios.request({
                    url: "https://bimayproxy.herokuapp.com/fetch/https://binusmaya.binus.ac.id/services/ci/index.php/student/init/getCourses",
                    method: "get",
                    headers: {
                        Bisquit: `PHPSESSID=${this.$store.state.phpsessid}`
                    }
                })
                    .then((response) => {
                        this.courses = response.data.Courses;
                        this.$Progress.finish();

                        this.getVideoConferences();
                        this.getAssignments();
                    })
                    .catch((error) => {
                        console.log(error);
                        this.$Progress.fail();
                    });
            },
            getVideoConferences() {
                this.$Progress.start();
                // https://binusmaya.binus.ac.id/services/ci/index.php/BlendedLearning/VideoConference/getList/ACCT6087/007392/1920/12252
                this.courses.forEach(e => {
                    const courseId = e.COURSEID;
                    const crseId = e.CRSE_ID;
                    const strm = e.STRM;
                    const classNbr = e.CLASS_NBR;

                    axios.request({
                        url: `https://bimayproxy.herokuapp.com/fetch/https://binusmaya.binus.ac.id/services/ci/index.php/BlendedLearning/VideoConference/getList/${courseId}/${crseId}/${strm}/${classNbr}`,
                        method: "get",
                        headers: {
                            Bisquit: `PHPSESSID=${this.$store.state.phpsessid}`
                        }
                    })
                        .then((response) => {
                            let data = response.data;

                            // remove stupid tag
                            data = data.replace(/<span.+vc="/g, "");
                            data = data.replace(/">.+>/g, "");

                            // convert html into json (stupid IT div.. huh!)
                            const classes = HtmlTableToJson.parse(`<table>${data}</table>`).results[0];

                            classes.forEach(e => {
                                e.classNbr = classNbr;

                                //split time
                                const time = e.Time.split(' - ');
                                e.startDate = `${e.Date} ${time[0]}`;
                                e.endDate = `${e.Date} ${time[1]}`;

                                this.videoConferences.push(e);

                                // sort
                                this.videoConferences = this.videoConferences.sort((a,b) => new moment(b.startDate) - new moment(a.startDate));
                            });

                            this.$Progress.finish();
                        })
                        .catch((error) => {
                            console.log(error);
                            this.$Progress.fail();
                        });
                });
                this.$Progress.finish();
            },
            getAssignments() {
                // https://binusmaya.binus.ac.id/services/ci/index.php/student/classes/assignmentType/COMP6229/011643/1920/LEC/14910/01
                this.$Progress.start();
                this.isLoading = true;
                this.courses.forEach(e => {
                    const courseId = e.COURSEID;
                    const crseId = e.CRSE_ID;
                    const strm = e.STRM;
                    const classNbr = e.CLASS_NBR;
                    const ssrComponent = e.SSR_COMPONENT;

                    axios.request({
                        url: `https://bimayproxy.herokuapp.com/fetch/https://binusmaya.binus.ac.id/services/ci/index.php/student/classes/assignmentType/${courseId}/${crseId}/${strm}/${ssrComponent}/${classNbr}/01`,
                        method: "get",
                        headers: {
                            Bisquit: `PHPSESSID=${this.$store.state.phpsessid}`
                        }
                    })
                        .then((response) => {
                            let data = response.data;

                            data.forEach((e, idx, array) => {
                                e.classNbr = classNbr;
                                this.assignments.push(e);

                                // insert into calendar
                                const calendarObj = {
                                    key: `${e.StudentAssignmentID}`,
                                    dot: 'purple',
                                    dates: new moment(`${e.deadlineDuration} ${e.deadlineTime}`, "DD MMM YYYY HH:mm:ss").toDate(),
                                    popover: {
                                        label: `${e.deadlineTime.substring(0,5)} - Asg. Deadline - ${this.courses.find(e => e.CLASS_NBR === classNbr).COURSENAME}`,
                                    },
                                };

                                this.calendarDates.push(calendarObj);

                                if (idx === array.length - 1){
                                    this.$Progress.finish();
                                    this.isLoading = false;

                                    // sort
                                    this.assignments = this.assignments.sort((a,b) => new moment(b.deadlineDuration) - new moment(a.deadlineDuration));
                                }
                            })
                        })
                        .catch((error) => {
                            console.log(error);
                            this.$Progress.fail();
                            this.isLoading = false;
                        });
                });
            },
            async getRandomQuote() {
                const DailyQuoteRepository = Repository.get("dailyQuotes");
                const { data } = await DailyQuoteRepository.get();
                this.randomQuote = data.contents.quotes[0].quote;
            },
            getClassSchedules() {
                // https://binusmaya.binus.ac.id/services/ci/index.php/student/class_schedule/classScheduleGetStudentClassSchedule
                this.$Progress.start();
                this.isLoading = true;

                axios.request({
                    url: "https://bimayproxy.herokuapp.com/fetch/https://binusmaya.binus.ac.id/services/ci/index.php/student/class_schedule/classScheduleGetStudentClassSchedule",
                    method: "get",
                    headers: {
                        Bisquit: `PHPSESSID=${this.$store.state.phpsessid}`
                    }
                })
                    .then((response) => {
                        this.classSchedules = response.data;
                        this.$Progress.finish();
                        this.isLoading = false;
                        this.insertDatesToCalendar();
                        this.getNextClass();
                    })
                    .catch((error) => {
                        console.log(error);
                        this.$Progress.fail();
                        this.isLoading = false;
                    });
            },
            insertDatesToCalendar() {
                this.$Progress.start();
                this.isLoading = true;

                // console.log(this.classSchedules.filter(c => {return c.ROOM === 400}))

                // input class schedules
                this.classSchedules.forEach(e => {
                    let dot = true;
                    if (e.N_DELIVERY_MODE === "GSLC") {
                        dot = "red";
                    } else if (e.N_DELIVERY_MODE === "F2F") {
                        dot = "orange";
                    } else if (e.N_DELIVERY_MODE === "VC") {
                        dot = "green";
                    }

                    const calendarObj = {
                        key: `${e.CRSE_CODE}-${e.SessionIDNum}`,
                        dot,
                        dates: [
                            { start: new moment(`${e.START_DT.substring(0,10)} ${e.MEETING_TIME_START}`, "YYYY-MM-DD HH:mm").toDate(),
                                end: new moment(`${e.END_DT.substring(0,10)} ${e.MEETING_TIME_END}`, "YYYY-MM-DD HH:mm").toDate() }
                        ],
                        popover: {
                            label: `${e.MEETING_TIME_START} - ${e.N_DELIVERY_MODE} - ${e.COURSE_TITLE_LONG} (${e.SSR_COMPONENT})`,
                        },
                    };

                    this.calendarDates.push(calendarObj);
                });
            },
            getNextClass() {
                let nextClasses = this.classSchedules.filter(c => {
                    return moment(c.END_DT.substring(0,10)).isAfter(moment())
                })

                let nextClass = nextClasses[0];
                nextClass.startDate = moment(`${nextClass.START_DT.substring(0,10)} ${nextClass.MEETING_TIME_START}`, "YYYY-MM-DD HH:mm").toDate();
                nextClass.endDate = moment(`${nextClass.END_DT.substring(0,10)} ${nextClass.MEETING_TIME_END}`, "YYYY-MM-DD HH:mm").toDate();

                if (moment(nextClass.startDate).isSame(moment(), 'day')
                    || moment(nextClass.startDate).isSame(moment().add(1,'days'), 'day')) {
                    this.nextClass = nextClass;
                }
            },
        },
        computed: {
            greetings: function() {
                const today = new Date()
                const curHr = today.getHours()

                if (curHr < 12) {
                    return "Good morning"
                } else if (curHr < 18) {
                    return "Good afternoon"
                } else {
                    return "Good evening"
                }
            }
        },
        filters: {
            relativeTime: function (date, format) {
                // console.log(date + " " + moment(date, "DD MMM YYYY HH:mm:ss").fromNow())
                return moment(date, format).fromNow();
            },
            lowerCase: function (string) {
                return string ? string.toLowerCase() : string;
            }
        },
        mounted() {
            this.checkSession();
            this.getBinusianData();
            this.getCourses();
            this.getRandomQuote();
            this.getClassSchedules();
        }
    }
</script>