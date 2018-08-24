---
title: Mapping request parameters in dynamically client generated fieldsets
     onto Java Beans
---

To have a good user experience with HTML forms we often need to let users
generate additional form fields client side.  If the user needs to be
able to enter the names of their favorite dishes, we just need to have a button 
which clones the appropriate input field with JavaScript.  Because HTML 
forms naturally support packing multiple request parameter values for a
single parameter name in what in most languages HTTP APIs ends
up presented as some sort of an array or list, this is trivial and easy to
process.

Sometimes, these additional fields need to be logically (and visually)
grouped as they represent a single complex object --- in Java this will
usually be a (persisted) Java Bean, where every relevant bean field needs to map to a
form input element.  Since HTML forms (or HTTP requests) have no ability to nest
parameters and since it seems support for this case is not usually included either natively or in web libraries/frameworks,
processing of such forms requires a custom solution.[^1]  What 
motivated this article was that I could not find a single JavaScript snippet on the
web that correctly implemented client side functionality, although a couple came
very close and enabled me, an almost total JS ignorant, to come up with a 
working one fairly quickly.  So here is a simple solution using JavaScript on the
client to add HTML fieldsets and process them in a plain Java servlet using a
little reflection on the server side.


## Example model

Assume we have a very simple strength training tracker where users can define
particular training programs, and add an arbitrary number of exercises that are
part of the program.  For this to be a reasonable user experience
in the early 21<sup>st</sup> century it should be possible to enter details of
each exercise in the same form.  Although fields belonging to each book will be
visually grouped inside an HTML fieldset depending on your browser or CSS framework, which makes semantic and UI sense, this
has no protocol meaning and each such field represents an independent POST
parameter.

The relevant part of our domain model consists of two Beans, `Program` and
`ProgramExercise` (with JPA annotations for completeness and clarity):

{% highlight java linenos %}
public class Program {
    @Id
    @GeneratedValue
    private int id;

    private String name;
    private String comment;

    @ManyToOne
    private User user;

    @OneToMany(mappedBy = "program", cascade = CascadeType.ALL)
    private List<ProgramExercise> exercises = new ArrayList<>();
    
    // Other fields, constructors, getters, setters, etc.
}
{% endhighlight %}

{% highlight java linenos %}
public class ProgramExercise {
    @Id
    @GeneratedValue
    private int id;

    @ManyToOne
    private Program program;

    private String exerciseName;
    private int exerciseOrder;
    private int sets;
    private int reps;
    
    // Other fields, constructors, getters, setters, etc.
}
{% endhighlight %}


## HTML and JavaScript

Below we have a simple input form.

{% highlight html linenos %}
<form action="/add-program" method="POST">
    <label>Program name
        <input type="text" name="program-name">
    </label>
    <label>Comment
        <textarea name="program-comment"></textarea>
    </label>

    <div id="fieldset-container">
        <fieldset id="exercise-fieldset">
            <legend>Exercise</legend>

            <label>Exercise name
                <input type="text" name="0-exercise-name">
            </label>

            <label>Exercise order
                <input type="number" name="0-exercise-order">
            </label>

            <label>Number of sets
                <input type="number" name="0-exercise-sets">
            </label>

            <label>Number of reps
                <input type="number" name="0-exercise-reps">
            </label>
        </fieldset>
    </div>

    <button type="button" onclick="cloneFieldset();">Add exercise</button>

    <input type="submit" value="Save">
</form>
{% endhighlight %}

Fields belonging to `ProgramExercise` are enclosed in an fieldset which serves
to both visually and semantically group them and as a container to clone.
The fieldset is enclosed in a `div#fieldset-container` which serves as a
container to append cloned fieldsets to.
Once the user clicks on the button to add another exercise, the following
JavaScript does the work.

{% highlight javascript linenos %}
var _counter = 1;

function cloneFieldset() {
    var container = document.getElementById("fieldset-container");
    var fieldset = document.getElementById("exercise-fieldset");
    var fieldsetClone = fieldset.cloneNode(true);
    var fieldsetChildren = fieldsetClone.children;

    var re = /^(0)(-.*)/;

    for (var i = 0; i < fieldsetChildren.length; i++) {
        childName = String(fieldsetChildren[i].name);
        fieldsetChildren[i].name = childName.replace(re, _counter + "$2");
    }
    _counter++;
    container.appendChild(fieldsetClone);
}
{% endhighlight %}

We clone the fieldset and get an array (line 7) of it children elements.  When
using CSS frameworks, due to various divs they need, some care needs to be taken in order to have proper
nesting of HTML elements --- for this example to work, form input elements need to
be direct children of the cloned fieldset, otherwise the JS snippet shown needs
to be adjusted.

On line 9 we define the regular expression to catch the names of relevant form
fields.  Only the initial digit will change, so we remember the part after it
(the digit part doesn't really need to be remembered in this case, but we do so
for clarity).  We then loop through the children and replace each child's name
with current counter value and the second part of our regular expression.
Finally, we increment the counter and append the new fieldset to the top
container.


## The controlling servlet

The heart of the solution is the servlet which initializes `ProgramExercise`
objects by mapping request parameter names to object setters, and then
invokes them using introspection. WHATWG HTML5 spec doesn't seem to
explicitly guarantee that the parameter order will stay the same as the order
in the form so we cannot rely on this order for example to avoid
introspection and looking into parameter names. Besides, relying on the order
of parameters would be a really bad idea in a system with so many moving
parts (even if HTML guaranteed such a thing, does the Java Servlet API,
etc.).

{% highlight java linenos %}
@WebServlet("/add-program")
public class AddProgramServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        User user = (User) request.getSession().getAttribute("user");

        Map<String, String[]> requestParameters = request.getParameterMap();

        String programName = requestParameters.get("program-name")[0];
        String programComment = requestParameters.get("program-comment")[0];

        Map<String, String> parameterMethodMap = new HashMap<>();
        parameterMethodMap.put("name", "setExerciseName");
        parameterMethodMap.put("order", "setExerciseOrder");
        parameterMethodMap.put("sets", "setSets");
        parameterMethodMap.put("reps", "setReps");

        Map<String, ProgramExercise> parameterToProgramExercises = new HashMap<>();
        String exerciseRe = "^\\d+-exercise-.+$";

        for (Map.Entry<String, String[]> parameter : requestParameters.entrySet()) {
            String[] parameterParts;
            String metodName;

            if (parameter.getKey().matches(exerciseRe)) {
                parameterParts = parameter.getKey().split("-");
                metodName = parameterMethodMap.get(parameterParts[2]);
            } else {
                continue;
            }

            if (!parameterToProgramExercises.containsKey(parameterParts[0])) {
                ProgramExercise exercise = new ProgramExercise();
                parameterToProgramExercises.put(parameterParts[0], exercise);
                invokeMethod(exercise, parameter.getValue()[0], metodName);
            } else {
                ProgramExercise exercise = parameterToProgramExercises.get(parameterParts[0]);
                invokeMethod(exercise, parameter.getValue()[0], metodName);
            }
        }
        List<ProgramExercise> programExercises = new ArrayList<>(parameterToProgramExercises.values());

        Program program = new Program(programName, programComment, programIsActive, user, programExercises);

        // Here we can persist the Program object and do other actions as needed.
    }

    private void invokeMethod(ProgramExercise exercise, String paramValue, String methodName) {
        try {
            Method method = exercise.getClass().getMethod(methodName, String.class);
            method.invoke(exercise, paramValue);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}

Lines 14--18 define parameter name to setter mapping.  Only the last part of
the parameter name is relevant here, so we use only it.  In order to process the
list of parameters in a single pass we will hold `ProgramExercise` objects in
a map declared on line 20.  We use the digit at the beginning of parameter names
to identify each program exercise.  Line 21 defines the regular expression we use
to filter the entry set of request parameters for the relevant ones on line 27.

The loop beginning on line 23 does all the work.  If the parameter refers to
a new exercise, initialize it, make a new entry in `parameterToProgramExercises` map,
and invoke the needed setter on the new exercise.  If the exercise already exists
in the map, get it and invoke the setter for the parameter in question.  The
`invokeMethod` helper method does the introspection and invocation work.

Finally, once all the parameters are processed we extract the exercises from the
working map, construct the new `Program` object, persist it, and do whatever
else we may need to.

* * *

[^1]: PHP [requires “`[]`” elements at the end of a parameter name in order to recognize it as having an array value](https://stackoverflow.com/a/11786558).
    Intentionally or not,
    [multidimensional arrays are supported with this syntax](https://stackoverflow.com/a/1576861).
    In Python land, [Django formsets](https://docs.djangoproject.com/en/dev/topics/forms/formsets/)
    provide server side support for more or less exactly the problem this article is
    discussing.  In Java EE plain servlet API does not have this
    supported (considering it is a low level API we can't complain),
    but neither do poplar frameworks like Spring and others as far as I was able
    to determine in a few bouts of googling.
