---
layout: single
title: "Unit test AWS Lambda in Go"
date: 2020-05-17
permalink: /unit-test-aws-lambda-in-go/
categories: aws-lambda go unit-testing
excerpt: "How to do unit test AWS lambda in Go."
---
{% include toc %}

# Introduction 

When I started working in Go and AWS Lambda, one of the difficulties that I faced was unit testing. I had a decent idea about what is unit testing and knew how to do it in Ruby but in Go, I had no idea because I was a beginner.

Learning Go was itself a challenge. Mainly because Go isn't a OOP language. I started reading articles on Go and watched numerous video series on YouTube.

After a couple of days, I was getting better, and I was able to understand things. But I wanted to learn how to do unit testing and unfortunately, there aren’t many good blogs that explain how to do unit test AWS services using Go. So this blog is an effort to explain how to unit test properly AWS services using Go.

In the blog, I will demonstrate how to unit test a lambda which uses EMR service in Go. The code is simple which is given a cluster id I have to get the cluster status.

Always remember if you want to do unit testing in Go then you have to use interface and avoid using concrete API or function as far as possible. So, for aws-sdk-go, we have interfaces for example for dynamodb we have
dynamodbiface You can see [aws-sdk-go](https://docs.aws.amazon.com/sdk-for-go/api/) to see what is the iface name of the service. Normally, its service-nameiface.

## Code

Now let’s start the coding
First I will create structs which will have cluster id as an input and emr interface as an API

```go
// ClusterInput represent input which will be given to the lambda
type ClusterInput struct {
 ClusterID string `json:"clusterID"`
}

// awsService represents emr interface
type awsService struct {
 emr emriface.EMRAPI
}
```

Next, I will create a function whose job will be to create a new AWS session and create a new emr service

```go
// newAWSService returns a new instance of emr
func newAWSService() *awsService {
 awsConfig := &aws.Config{Region: aws.String("us-west-2")}
 sess, err := session.NewSession(awsConfig)
 if err != nil {
  log.Errorf("error while creating AWS session - %s", err.Error())
 }

 return &awsService{
  emr: emr.New(sess),
 }
}
```

Now, comes the meaty part. I will do input validation and prepare input to the DescribeCluster emr API method. Rest all is simple.

```go
// getClusterStatus returns current cluster status along with an error
func (svc *awsService) getClusterStatus(input ClusterInput) (string, error) {
 clusterID := input.ClusterID
 if clusterID == "" {
  return "", errors.New("clusterID is empty")
 }

 describeClusterInput := &emr.DescribeClusterInput{
  ClusterId: aws.String(clusterID),
 }

 clusterDetails, err := svc.emr.DescribeCluster(describeClusterInput)
 if err != nil {
  log.Errorf("DescribeCluster error - %s", err)
  return "", err
 }

 if clusterDetails == nil {
  log.Errorf("clusterID does not exist")
  return "", errors.New("clusterID does not exist")
 }

 clusterStatus := *clusterDetails.Cluster.Status.State

 return string(clusterStatus), nil
}
```

The important point to see is how I used &emr on DescribeClusterInput.. If you want to use any other AWS service then you should be doing something similar.

## Unit Testing

Let's start testing now

For testing, I will be using [stretchr/testify](https://github.com/stretchr/testify) because it provides [mock](https://github.com/stretchr/testify#mock-package) and [assert](https://github.com/stretchr/testify#assert-package) functionality. Especially mock is very important. When you write unit tests then its essential that it should not call the real service. It should always call mock service.

Here first, I will create mock emr and create mock implementation of DescribeCluster method. After that I will create setup method

```go
// mockEMR represents mock implementation of AWS EMR service
type mockEMR struct {
 emriface.EMRAPI
 mock.Mock
}

// DescribeCluster is a mocked method which return the cluster status
func (m *mockEMR) DescribeCluster(input *emr.DescribeClusterInput) (*emr.DescribeClusterOutput, error) {
 args := m.Called(input)
 return args.Get(0).(*emr.DescribeClusterOutput), args.Error(1)
}

func setup() (*mockEMR, *awsService) {
 mockEMRClient := new(mockEMR)
 mockEMR := &awsService{
  emr: mockEMRClient,
 }

 return mockEMRClient, mockEMR
}
```

Now, it's time to write [table driven tests](https://github.com/golang/go/wiki/TableDrivenTests) and call original function. Once the original function is called then I can assert whether expected result match with the actual result or not.

```go
mockEMRClient, mockEMR := setup()

mockDescribeClusterInput := &emr.DescribeClusterInput{
 ClusterId: aws.String(testCase.clusterID),
}

mockDescribeClusterOutput := &emr.DescribeClusterOutput{
 Cluster: &emr.Cluster{
  Status: &emr.ClusterStatus{
   State: aws.String(testCase.expectedClusterStatus),
  },
 },
}

mockEMRClient.On("DescribeCluster", mockDescribeClusterInput.Return(mockDescribeClusterOutput, testCase.emrError)
res, err := mockEMR.getClusterStatus(testCase.expectedInput)

assert.Equal(t, testCase.expectedClusterStatus, res, testCase.message)
assert.IsType(t, testCase.expectedError, err, testCase.message)
```

That's it!  

## Repo

I hope after reading this blog you can understand little-bit of unit testing AWS lambdas in Go.
Checkout [aws-unit-test-golang](https://github.com/belikemike21/aws-unit-test-golang) for the full code.


*Jacky*
