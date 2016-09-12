---
layout: post
title: Faster data retrieval in Rails
excerpt: "How to skip the Rails magic to get your data faster."
tags: [ruby, rails, activerecord, performance]
comments: true
share: false
---

Rails and Active Record provide plenty of magic to get web applications up and running quickly, which is great. Associations make it easy to write short, readable code that produces complex SQL behind the scenes. That magic can also lead to some inefficiencies that slow down your app. I'll try to explain why and how to do it better in this post.

A common task in Rails is to retrieve data and return it as JSON to your front end. This can be easy in Rails, but if you're not careful you can run into a couple problems. One is that Active Record magic can lead to running too many inefficient queries. The other less obvious problem is that <i>instantiating Active Record objects is expensive.</i> I've put together a little example to demonstrate this and to show how to improve some typical Rails code. 

For this demo, I'm going to create a new Rails app, create some models, then use the Rails console to insert sample data and measure how long it takes to retrieve the data.

### Setup App and Models

First, let's create the app and models. We'll build a little school system.

{% highlight bash %}
$ rails new performance_test

$ rails generate model Student name:string
$ rails generate model Classroom name:string
$ rails generate model Subject name:string
$ rails generate model Teacher name:string
$ rails generate model Course subject:belongs_to teacher:belongs_to classroom:belongs_to
$ rails generate model StudentCourse course:belongs_to student:belongs_to

$ rake db:migrate
{% endhighlight %}

Our school has Students, Classrooms, Subjects and Courses. A Course has one subject, classroom and teacher, and many students. For the sake of keeping this short, I've stuck with the default SQLite database.

Let's add some extra associations to the Course model to indicate that a Course has many Students:

{% highlight ruby %}
class Course < ActiveRecord::Base
  belongs_to :subject
  belongs_to :teacher
  belongs_to :classroom
  has_many :student_courses
  has_many :students, through: :student_courses
end
{% endhighlight %}

Now let's populate some data. We'll create a School class to contain our helper methods to create data and run the tests.

{% highlight ruby %}
class School
  def self.create_sample_school
    # delete any existing data
    StudentCourse.destroy_all
    Course.destroy_all
    Classroom.destroy_all
    Teacher.destroy_all
    Student.destroy_all

    # create 100 classrooms, teachers, subjects and courses with 20 students in each course
    100.times do |n|
      classroom = Classroom.create! name: "Classroom#{n}"
      teacher = Teacher.create! name: "Teacher#{n}"
      subject = Subject.create! name: "Subject#{n}"
      course = Course.create! subject_id: subject.id, teacher_id: teacher.id, classroom_id: classroom.id
      
      20.times do |x|
        student = Student.create! name: "Student#{(n*20) + x}"
        StudentCourse.create! course_id: course.id, student_id: student.id
      end
    end
  end
end
{% endhighlight %}

The `create_sample_school` method creates 100 courses, each with a classroom, teacher and subject, and 2000 students (20 per course). This gives us a decent set of data to do some performance testing with. Let's run it in the rails console.

{% highlight bash %}
$ rails c
{% endhighlight %}
{% highlight ruby %}
> School.create_sample_school
{% endhighlight %}

### Do some performance testing

Let's pretend our school needs to print out all the course information and student listing on the school's bulletin board. They decided to do this by logging each course with its subject, teacher, classroom and student list to the console. Here's our method for doing that:

{% highlight ruby %}
def self.print_school_slowly
  start = Time.now
  all_courses = Course.all

  all_courses.each do |course|
    puts "*******"
    puts "Subject:"
    puts course.subject.name
    puts "Teacher:"
    puts course.teacher.name
    puts "Classroom:"
    puts course.classroom.name

    puts "Students:"
    course.students.each {|student| puts student.name}
  end

  puts "Total time: #{Time.now - start}"

  return
end
{% endhighlight %}

I like to measure how long a ruby method runs by simply comparing the time at the start and end. The `print_school_slowly` method uses standard Active Record associations to get the data we need. Let's run it a few times:

{% highlight bash %}
> School.print_school_slowly
> ... school gets printed ...
> Total time: 0.286196
> School.print_school_slowly
> Total time: 0.260675
> School.print_school_slowly
> Total time: 0.246969
{% endhighlight %}

1/4 of a second to retrieve the data... Remember that's being added on to the time a user is waiting for a page to load, and a typical web application does much more complex stuff than this. This is too slow. Let's speed it up.

A well-known way to speed this up is to use `includes`. This will introduce eager loading of the associations so we don't have to run additional queries behind the scenes.

{% highlight ruby %}
def self.print_school_less_slowly
  start = Time.now
  all_courses = Course.includes(:subject, :teacher, :classroom)

  all_courses.each do |course|
    puts "*******"
    puts "Subject:"
    puts course.subject.name
    puts "Teacher:"
    puts course.teacher.name
    puts "Classroom:"
    puts course.classroom.name

    puts "Students:"
    course.students.each {|student| puts student.name}
  end

  puts "Total time: #{Time.now - start}"
  return
end
{% endhighlight %}

{% highlight bash %}
> School.print_school_less_slowly
> ... school gets printed ...
> Total time: 0.188844
> School.print_school_less_slowly
> Total time: 0.149944
> School.print_school_less_slowly
> Total time: 0.15106
{% endhighlight %}

We've improved by 0.1 seconds, nice. We can do better, though. For each course, we're instantiating the Course, Subject, Teacher and Classroom objects and 20 Student objects. Like I said earlier, instantiating ActiveRecord objects is expensive. In this case, we don't really use them for anything other than grabbing a string property. Instead, we can use ActiveRecord's `pluck` method. It returns the fields from the database without instantiating ActiveRecord objects.

{% highlight ruby %}
def self.print_school_faster
  start = Time.now

  all_courses = Course.includes(:subject, :teacher, :classroom)
    .pluck("courses.id", "subjects.name", "teachers.name", "classrooms.name")

  all_courses.each do |course|
    puts "*******"
    puts "Subject:"
    puts course[1]
    puts "Teacher:"
    puts course[2]
    puts "Classroom:"
    puts course[3]

    puts "Students:"
    student_names = StudentCourse.where(course_id: course[0]).includes(:student).pluck("students.name")
    student_names.each {|s| puts s}
  end

  puts "Total time: #{Time.now - start}"
end
{% endhighlight %}

{% highlight bash %}
> School.print_school_faster
> ... school gets printed ...
> Total time: 0.072936
> School.print_school_faster
> Total time: 0.075538
> School.print_school_faster
> Total time: 0.068242
{% endhighlight %}

We've cut the time in half again. That was all time being spent instantiating ActiveRecord objects. 

Finally, I'm not happy that we are still querying the student_courses table 100 times (once per course). Let's prevent that. This is where the code starts getting a little uglier, so you have to decide whether you'd rather trade off performance for elegance. In my opinion, performance almost always wins.

{% highlight ruby %}
def self.print_school_fastest
  start = Time.now

  all_courses = Course.includes(:subject, :teacher, :classroom)
    .pluck("courses.id", "subjects.name", "teachers.name", "classrooms.name")

  student_course_lookup = {}
  StudentCourse.includes(:student).pluck(:course_id, "students.name").each do |sc| 
    if student_course_lookup[sc[0]].blank? 
      student_course_lookup[sc[0]] = [sc[1]]
    else
      student_course_lookup[sc[0]] << sc[1]
    end
  end

  all_courses.each do |course|
    puts "*******"
    puts "Subject:"
    puts course[1]
    puts "Teacher:"
    puts course[2]
    puts "Classroom:"
    puts course[3]

    puts "Students:"
    student_names = student_course_lookup[course[0]]
    student_names.each {|s| puts s}
  end

  puts "Total time: #{Time.now - start}"
end
{% endhighlight %}

{% highlight bash %}
> School.print_school_fastest
> ... school gets printed ...
> Total time: 0.018738
> School.print_school_fastest
> Total time: 0.018821
> School.print_school_fastest
> Total time: 0.019621
{% endhighlight %}

Now we're down to about 0.019 seconds. What I did here was instead of querying the student_courses table for each course, I just ran one query to get all students before the loop and built up a hash where the keys are course IDs and the values are arrays of student names. Pulling those from memory in the loop is faster than hitting the database every time. 

The final version is about 15 times faster than the first version. To summarize how we did it, we reduced the number of database queries and reduced the number of Active Record objects we instantiated. In real life scenarios, these techniques can result in huge performance boosts. You can get better page load times and process backgrounds jobs faster. Even though the code is a little uglier, it's well worth making a user's experience better.