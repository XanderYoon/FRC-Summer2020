# InfiniteRecharge (Summer 2020)
JCHS Gladiator Robotics Team's Github Repository for 2020 season (Infinite Recharge Game)

# Autonomous Motor Control
Closed Loop Motor Control with the Talon SRX and Quadrature Encoders

## Chapter 1: Introduction
Our motor controller of choice is the Talon SRX from Cross the Road Electronics™ (CTRE).  Until now we’ve been using the Talon as a simple voltage controller- giving our motor -100% to 100% of the battery voltage in order to induce a speed to power to the motor.   That’s great for Teleop, and your human driver becomes the feedback loop.

Our objectives:
1. Drive straight forward to an exact distance and stop
2. Turn to a specific angle
   

## Chapter 2: PID loops for position control
### P term:
PID is an acronym, it stands for ‘Proportional, Integral, differential’.  Let’s run through these one at a time.

Let’s say you’re at position 10, and you WANT to get to position 500.  That means we have 490 to go until we get there.  490 what?!  What are we talking about?  It doesn’t matter.  Could be inches, could be encoder ticks.  As long as your unit never changes, use whatever is convenient. 

So getting back on point, we’re at 10.  Our goal is 500.  That makes our Error = (goal - position) = 490. Proportional control (just the P in PID) can be realized as follows:

Control = Kp * Error  (remember Error = goal - position).

Kp is a ‘Constant’ or a real number that we set to be our proportional term.  It’s part of what we tune to get accurate control.  Right now our Talon SRX can be set from -1023 to +1023, which is full reverse to full forward (0 is stop).  If we set our Kp to, say, 1, our current motor power will be 490 (see what I did there?  Motor power = Kp * Error.)  This might be fine if there’s 1000 counts per revolution.  It may be way too little if there’s only 5 counts per revolution, and your motor will slow to a stop long before you’ve reached your goal.


### I term:
The I, or Integral term is the first term that “remembers” some of our error since the last time we ran an error.  Let’s continue on our previous example:  I’m now at 495 counts, and I want to get to 500.  In this case, my Error = (500-495) = 5.  OK, I’m only 5 away from  my goal.  Is that good enough?  Well, it had better be, because with Proportional control (remember above we set our P term to 1) our motor power is only going to be 5.  5/1023 isn’t going to move the motor alone let alone a gearbox and a 120 pound robot.  So what do we do?  This is where the I term really shines:

Integral term = Ki * (Error) + Integral term from last time through the loop

Here’s why it’s awesome:  If you’re close and stalled, using an integral term, you can slowly crank up the power you need in order to get that motor moving towards its final goal.

Here’s why it sucks:  The integral term can get out of hand VERY quickly.  The example above was MADE for an I term, but horrible if you’re at 5 and want to reach 20000.  The proportional term alone can get you to full power, so the I term just builds and builds each time with nowhere to go.  How do you handle that situation?  The answer is cheat.  Yep, this is your code, and you can do what you want with it.  Here’s some ways to deal with a runaway I term:
1. Make the I term = 0 every time your error changes sign (from + to - or vice versa).  This indicates that you’ve already crossed your goal, and the Iterm is driving you further away.
2. Don’t even use an Iterm unless you’re close enough that the Pterm won’t get you there.  This way, your Iterm can’t build unless it’s necessary.

### D Term
The D term, or differential term, is how we apply the brakes.  Up until now we’ve talked about how to GET to where we’re going, but our next challenge is to slow down in time to stop where we want.  Your D term is calculated as follows:

Differential Term = Kd * (Error Last time - Error now)

Looking at that, it’s a measure of how much we’ve IMPROVED on our position, and it subtracts power as we reach our goal.  This way the entire PID is determined as:

PID=P + I - D

## Chapter 3: Tuning for position
Let’s start by tuning P, the proportional term of our PID loop.
Tuning P
Start by making Kp=0.5.  Set a target with the Z axis and pull the switch.  Chances are pretty good it’s not going to get there by the time the motor grinds to a halt. What we’re looking for in this case is oscillation.  We want the motor to overshoot its goal, and swing back in the opposite direction.  If it’s doing this multiple times, perfect.  If not, double Kp and try again until it DOES do this.  Keep doubling P until you have a good oscillating starting point.

Tuning D
Remember our purpose for the D term is to SLOW down if we’re approaching the goal too quickly.  It prevents overshoot, and it seems like exactly the behavior we need right now.  Once you have an oscillation using only a P value, try setting your initial Kd to 10x to 100x of your P value.  NOW comes the tuning.  Has it stopped oscillating?  If not, increase your D until it does.  If not, reduce your D until it does, then add some back.

At this point, you’re just about critically damped.  Especially from a distance, even with just a P and a D term.  So why use an I term at all?  Well, when you’re pretty close to your goal, but you REALLY want to be RIGHT ON the goal, a PD loop might not cut it.  Before proceeding, try it now- set your goal to, say, 20 encoder clicks away from your current position (you can hand turn the wheel to achieve this).

Tuning I 
Our system is stable, it just needs a bit of oomph at the end to get to its destination. If you’re REALLY far away, you don’t even want an I term, because it’s just going to grow unreasonably, and when you get close to home, it’s all bulked up, and it’ll cause wild overshoot. The Talon has a built in function for JUST this occasion.  You can set an IntegralZone, which is the amount of error beyond which you clear the integral term.  

_motor.config_IntegralZone(0, IZone, 0);

That iZone in there iss an int, and you can set it in the same place you set your other parameters.  Set that to just a little more than your maximum error is when you’re driving with a PD.  Then set Ki (your I term) to about 0.001 of Kp and give it a try (a little goes a long way).  If it’s too slow to correct near home, that’s OK, try doubling Ki, or increase your Integral Zone a little and repeat until you’re satisfied.   PLEASE tune only one variable at a time.

## Chapter 4: Using PID for velocity control
Velocity control is most obviously used for a shooter.  For example: we need to spin a flywheel at a constant velocity, and if we’re rapid-firing, we need to supply the motor with more power so that it maintains it’s ideal speed for a consistent shot when we’re firing 1 object or 10 in a row.  What’s NOT so obvious is that velocity control is what you need to drive a tank straight.  Think this way:  In order to go straight, I really need to turn the drive on both sides at exactly the same distance, at exactly the same rate.  You also need to overcome things like extra friction, motor direction preference, and external obstacles.  So far we’ve been doing ‘Drive Straight’ by counting pulses, and giving the motor that falls behind extra power.  The biggest problem here is that we’re already behind, and we’ve noticed that when we try to go faster, it becomes unstable and we can’t correct for the error.
We need to react faster, but how?  If you connect your encoder directly to the Talon, it computes velocity at each encoder tick by measuring the exact time since the last one.  Additionally, it reads encoders at 4x the normal rate, so we’re getting velocity updates 1024 times per revolution of the encoder.  Now that we’ve got fast updates, why don’t we let the Talon handle the control as well?  All we have to do is set the rules for our controller to follow.  You’ve already read about the PID and our test setup, so let’s jump right into tuning.  Note that to apply this to a tank drive, the same tuning can apply to both sides.

New Constant:  Tuning F: Feed-forward.  
Feedback is monitoring your system, and making corrections when it’s not where you want it to be.  Feed FORWARD actually lets you make an adjustment BEFORE the disturbance happens. In our case, since we’re controlling velocity, we’re going to use feed-forward to set our approximate initial speed.  It’s a way to say “You want to go X velocity, I’m going to start by giving the motor X times Kf”  (Kf is your feed-forward constant).  

That velocity represents the number of encoder counts per 100ms.  Kind of makes sense, after a 7:1 gear ratio, it means that my encoder is spinning at (1500/1024) * 10 *60sec/1min = 878rpm.  The motor is running 878 * 7 = 6152 rpm. So what’s our Kf?  Easy- it is equal to power to the motor in native units (1023) divided by measured velocity (1500), or in this case: 1023/1500 = 0.682.  THAT is your Kf.  We just measured it.  Boom.

Tuning P
We’ve got an F, but all this will do is set a basic power level depending on how fast you want to go.  It won’t adjust accurately for every speed, and it won’t overcome unexpected friction or obstacles.  Let’s add in a little P (proportional) response to make up for the difference.

