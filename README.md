# GRAPHQL-SCHEMA
Graphql
# =========================
# SCALAR TYPES
# =========================
scalar Date
scalar JSON

# =========================
# ENUMS
# =========================
enum Gender {
  MALE
  FEMALE
  OTHER
}

enum CourseLevel {
  BEGINNER
  INTERMEDIATE
  ADVANCED
}

enum EnrollmentStatus {
  ACTIVE
  COMPLETED
  DROPPED
}

# =========================
# INTERFACES
# =========================
interface Person {
  id: ID!
  name: String!
  email: String!
  gender: Gender!
  createdAt: Date!
}

# =========================
# TYPES
# =========================
type Student implements Person {
  id: ID!
  name: String!
  email: String!
  gender: Gender!
  createdAt: Date!
  rollNumber: String!
  cgpa: Float
  enrollments: [Enrollment!]!
}

type Professor implements Person {
  id: ID!
  name: String!
  email: String!
  gender: Gender!
  createdAt: Date!
  specialization: String!
  courses: [Course!]!
}

type Course {
  id: ID!
  title: String!
  description: String!
  level: CourseLevel!
  credits: Int!
  professor: Professor!
  enrollments: [Enrollment!]!
}

type Enrollment {
  id: ID!
  student: Student!
  course: Course!
  status: EnrollmentStatus!
  enrolledAt: Date!
  grade: String
}

type Department {
  id: ID!
  name: String!
  head: Professor!
  courses: [Course!]!
}

type DashboardStats {
  totalStudents: Int!
  totalProfessors: Int!
  totalCourses: Int!
  totalEnrollments: Int!
}

# =========================
# INPUT TYPES
# =========================
input StudentInput {
  name: String!
  email: String!
  gender: Gender!
  rollNumber: String!
}

input CourseInput {
  title: String!
  description: String!
  level: CourseLevel!
  credits: Int!
  professorId: ID!
}

input EnrollmentInput {
  studentId: ID!
  courseId: ID!
}

# =========================
# QUERIES
# =========================
type Query {
  students: [Student!]!
  student(id: ID!): Student

  professors: [Professor!]!
  professor(id: ID!): Professor

  courses(level: CourseLevel): [Course!]!
  course(id: ID!): Course

  dashboardStats: DashboardStats!
}

# =========================
# MUTATIONS
# =========================
type Mutation {
  createStudent(input: StudentInput!): Student!
  updateStudentCGPA(id: ID!, cgpa: Float!): Student!
  deleteStudent(id: ID!): Boolean!

  createCourse(input: CourseInput!): Course!
  enrollStudent(input: EnrollmentInput!): Enrollment!
  updateEnrollmentStatus(id: ID!, status: EnrollmentStatus!): Enrollment!
}

# =========================
# SUBSCRIPTIONS
# =========================
type Subscription {
  studentEnrolled: Enrollment!
}
# NODE.JS APOLLO SERVER IMPLEMENTATION 
const { ApolloServer, gql, PubSub } = require('apollo-server');
const { v4: uuidv4 } = require('uuid');

const pubsub = new PubSub();
const STUDENT_ENROLLED = "STUDENT_ENROLLED";

const students = [];
const professors = [];
const courses = [];
const enrollments = [];

const resolvers = {
  Query: {
    students: () => students,
    student: (_, { id }) => students.find(s => s.id === id),
    professors: () => professors,
    courses: (_, { level }) =>
      level ? courses.filter(c => c.level === level) : courses,
    dashboardStats: () => ({
      totalStudents: students.length,
      totalProfessors: professors.length,
      totalCourses: courses.length,
      totalEnrollments: enrollments.length
    })
  },

  Mutation: {
    createStudent: (_, { input }) => {
      const newStudent = {
        id: uuidv4(),
        createdAt: new Date(),
        cgpa: 0,
        ...input
      };
      students.push(newStudent);
      return newStudent;
    },

    updateStudentCGPA: (_, { id, cgpa }) => {
      const student = students.find(s => s.id === id);
      if (!student) throw new Error("Student not found");
      student.cgpa = cgpa;
      return student;
    },

    deleteStudent: (_, { id }) => {
      const index = students.findIndex(s => s.id === id);
      if (index === -1) return false;
      students.splice(index, 1);
      return true;
    },

    createCourse: (_, { input }) => {
      const professor = professors.find(p => p.id === input.professorId);
      if (!professor) throw new Error("Professor not found");

      const newCourse = {
        id: uuidv4(),
        ...input,
        professor
      };
      courses.push(newCourse);
      return newCourse;
    },

    enrollStudent: (_, { input }) => {
      const student = students.find(s => s.id === input.studentId);
      const course = courses.find(c => c.id === input.courseId);

      const enrollment = {
        id: uuidv4(),
        student,
        course,
        status: "ACTIVE",
        enrolledAt: new Date()
      };

      enrollments.push(enrollment);
      pubsub.publish(STUDENT_ENROLLED, {
        studentEnrolled: enrollment
      });

      return enrollment;
    }
  },

  Subscription: {
    studentEnrolled: {
      subscribe: () => pubsub.asyncIterator([STUDENT_ENROLLED])
    }
  }
};

const server = new ApolloServer({
  typeDefs: gql(require('fs').readFileSync('./schema.graphql', 'utf8')),
  resolvers
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
# SAMPLE QUERY
query GetAllStudents {
  students {
    id
    name
    rollNumber
    cgpa
  }
}
# SAMPLE MUTATION
mutation CreateStudent {
  createStudent(input: {
    name: "Braj Kishor",
    email: "braj@gmail.com",
    gender: MALE,
    rollNumber: "BCA101"
  }) {
    id
    name
  }
}
# OUTPUT
