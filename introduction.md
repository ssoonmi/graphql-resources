# Intro to GraphQL

GraphQL is a way for us to gain access to information in our database without relying on understanding what data each endpoint in our RESTful routes returns. 

As you may have realized from your Full Stack and MERN projects, you need to know exactly what data is being returned or sent up from each route in your backend to make use of that data. If your backend routes ever changed, you needed to make sure the frontend data fetched from those routes was still being formatted correctly. 

GraphQL allows the frontend to determine what data it wants from the backend instead of being at the disposal of whatever your backend sends up. With RESTful routes, you need to know which routes return which data. With GraphQL, your frontend can ask for only the information it needs from a single route.