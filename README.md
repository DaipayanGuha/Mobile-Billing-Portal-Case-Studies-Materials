# Mobile-Billing-Portal-Case-Studies-Materials
Steps for creating Mobile Billing Portal from Payroll Portal


package com.cg.mobile.services;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.cg.mobile.beans.Address;
import com.cg.mobile.beans.Bill;
import com.cg.mobile.beans.Customer;
import com.cg.mobile.beans.Plan;
import com.cg.mobile.beans.PostpaidAccount;
import com.cg.mobile.daoservices.BillDAO;
import com.cg.mobile.daoservices.CustomerDAO;
import com.cg.mobile.daoservices.PlanDAO;
import com.cg.mobile.daoservices.PostpaidAccountDAO;
import com.cg.mobile.exceptions.BillDetailsNotFoundException;
import com.cg.mobile.exceptions.CustomerDetailsNotFoundException;
import com.cg.mobile.exceptions.InvalidBillMonthException;
import com.cg.mobile.exceptions.PlanDetailsNotFoundException;
import com.cg.mobile.exceptions.PostpaidAccountNotFoundException;

@Component("billingServices")
public class BillingServicesImpl implements BillingServices {
	
	@Autowired
    private BillDAO billDAO;
	
	@Autowired
	private CustomerDAO customerDAO;
	
	@Autowired
	private PlanDAO planDAO;
	
	@Autowired
	private PostpaidAccountDAO postpaidAccountDAO;
    
    @Override
	public List<Plan> getAllPlanDetails() {
		return planDAO.findAll();
	}
    
	@Override
	public int acceptCustomerDetails(String firstName, String lastName, String emailID, String dateOfBirth,
			String billingAddressCity, String billingAddressState, int billingAddressPinCode, String homeAddressCity,
			String homeAddressState, int homeAddressPinCode) {
		Address homeAddress= new Address(homeAddressCity,homeAddressState,homeAddressPinCode);
		Address billingAddress=new Address(billingAddressCity,billingAddressState,billingAddressPinCode);
		List<Address> address=new ArrayList<Address>();
		address.add(homeAddress);
		address.add(billingAddress);
        Customer customer= new Customer(firstName, lastName, emailID, dateOfBirth, address);	
        customerDAO.save(customer);
		return customer.getCustomerID();
	}
	@Override
	public long openPostpaidMobileAccount(int customerID, int planID)
			throws PlanDetailsNotFoundException, CustomerDetailsNotFoundException {
		Customer customer = customerDAO.findById(customerID).orElseThrow(()->  new CustomerDetailsNotFoundException("Customer Details Not found for ID. : "+customerID));
		if(customer==null)
			throw new CustomerDetailsNotFoundException("Sorry, Customer Not Found!");
	     Plan plan = getPlanDetails(planID);
	     if(plan ==null)
	     throw new PlanDetailsNotFoundException("Please enter correct plan id.");
	     customer= getCustomerDetails(customerID);
		 PostpaidAccount postPaidAccount = new PostpaidAccount(plan, customer);
		 postpaidAccountDAO.save(postPaidAccount);
		 return postPaidAccount.getMobileNo();
	}
	@Override
	public double generateMonthlyMobileBill(int customerID, long mobileNo, String billMonth, int noOfLocalSMS,
			int noOfStdSMS, int noOfLocalCalls, int noOfStdCalls, int internetDataUsageUnits)throws CustomerDetailsNotFoundException,PostpaidAccountNotFoundException{
        int centralGST=9;
		int stateGST=9;
		Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		Map<Long, PostpaidAccount> postPaidAccount = customer.getPostpaidAccounts();
		if(postPaidAccount==null)
		throw new PostpaidAccountNotFoundException("Customer has no plans");
		PostpaidAccount postpaidAcc=postPaidAccount.get(mobileNo);		
        Bill bill = new Bill(noOfLocalSMS, noOfStdSMS, noOfLocalCalls, noOfStdCalls, internetDataUsageUnits, billMonth, stateGST, centralGST,postpaidAcc);
		float amount=0.0f;
		Plan plan =postpaidAcc.getPlan();	
		if(noOfLocalCalls>plan.getFreeLocalCalls()) {
			bill.setLocalCallAmount((noOfLocalCalls-plan.getFreeLocalCalls())*plan.getLocalCallRate());
			amount =amount+bill.getLocalCallAmount();
		}
		if(noOfLocalSMS>plan.getFreeLocalSMS()) {
			bill.setLocalSMSAmount((noOfLocalSMS-plan.getFreeLocalSMS())*plan.getLocalSMSRate());
			amount=amount+bill.getLocalSMSAmount();
		}
		if(noOfStdCalls>plan.getFreeStdCalls()) {
			bill.setStdCallAmount((noOfStdCalls-plan.getFreeStdCalls())*plan.getStdCallRate());
			amount=amount+bill.getStdCallAmount();
		}
		if(noOfStdSMS>plan.getFreeStdSMS()) {
			bill.setStdSMSAmount((noOfStdSMS-plan.getFreeStdSMS())*plan.getStdSMSRate());
			amount=amount+bill.getStdSMSAmount();
		}
		if(internetDataUsageUnits>plan.getFreeInternetDataUsageUnits()) {
			bill.setInternetDataUsageAmount((internetDataUsageUnits-plan.getFreeInternetDataUsageUnits())*plan.getInternetDataUsageRate());
		    amount=amount+bill.getInternetDataUsageAmount();
		}
		amount=amount+plan.getMonthlyRental()+(amount*stateGST)/100+(amount*centralGST)/100;
		bill.setTotalBillAmount(amount);
		billDAO.save(bill);
        return amount; 
	}
	@Override
	public Customer getCustomerDetails(int customerID) throws CustomerDetailsNotFoundException {
		if(customerDAO.findById(customerID)==null)
			throw new CustomerDetailsNotFoundException();
	   return customerDAO.findById(customerID).orElseThrow(()-> new CustomerDetailsNotFoundException());
	}
	@Override
	public List<Customer> getAllCustomerDetails() {
		return customerDAO.findAll();
	}
	@Override
	public PostpaidAccount getPostPaidAccountDetails(int customerID, long mobileNo)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException{
		Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer with this id found");
		 if(!customer.getPostpaidAccounts().containsKey(mobileNo))
			 throw new PostpaidAccountNotFoundException("Customer has no plans");
		 PostpaidAccount postpaidAccount = postpaidAccountDAO.findById((int)mobileNo).orElseThrow(()->  new PostpaidAccountNotFoundException("Postpaid Account Details Not found."));
		return postpaidAccount;
	}
	
	@Override
	public Bill getMobileBillDetails(int customerID, long mobileNo, String billMonth)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException, InvalidBillMonthException,
			BillDetailsNotFoundException, PlanDetailsNotFoundException {
		Customer customer = customerDAO.findById(customerID).orElseThrow(()->  new CustomerDetailsNotFoundException("Customer Details Not found for ID. : "+customerID));
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		PostpaidAccount postpaidAccount = postpaidAccountDAO.findById((int)mobileNo).orElseThrow(()->  new PostpaidAccountNotFoundException("Postpaid Account Details Not found."));
		Plan plan = postpaidAccount.getPlan();
		if (plan == null)
			throw new PlanDetailsNotFoundException("No Plan Details Found!");
		Bill bill =billDAO.findParticularBill(mobileNo, billMonth);
		return bill;
	}
	@Override
	public List<PostpaidAccount> getCustomerAllPostpaidAccountsDetails(int customerID)
			throws CustomerDetailsNotFoundException {
		Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		return postpaidAccountDAO.findAll();
	}
	
	@Override
	public List<Bill> getCustomerPostPaidAccountAllBillDetails(int customerID, long mobileNo)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException {
		Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		Map<Long, PostpaidAccount> postPaidAccount = customer.getPostpaidAccounts();
		if(postPaidAccount==null)
		throw new PostpaidAccountNotFoundException("Customer has no plans");
		//PostpaidAccount postpaidAcc=postPaidAccount.get(mobileNo);		
		//return billDAO.findAllById((int)mobileNo));
	}
	@Override
	public boolean changePlan(int customerID, long mobileNo, int planID)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException, PlanDetailsNotFoundException {
		Plan plan = getPlanDetails(planID);
		if(plan==null)
			throw new PlanDetailsNotFoundException("No Plan with this Id exist");
		PostpaidAccount postpaidAccount =getPostPaidAccountDetails(customerID, mobileNo);
		postpaidAccount.setPlan(plan);
		postpaidAccountDAO.save(postpaidAccount);
		return true;
	}
    @Override
	public boolean closeCustomerPostPaidAccount(int customerID, long mobileNo)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException {
		Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		if(customer.getPostpaidAccounts().containsKey(mobileNo))
		   postpaidAccountDAO.deleteById((int)mobileNo);
		 else throw new PostpaidAccountNotFoundException("Customer has no plans");
		 return true;
	}
    @Override
	public boolean removeCustomerDetails(int customerID) throws CustomerDetailsNotFoundException {
    	Customer customer = getCustomerDetails(customerID);
		if(customer==null)
		throw new CustomerDetailsNotFoundException("Sorry no customer found");
		customerDAO.deleteById(customerID);
		return true;
	}
    @Override
	public Plan getCustomerPostPaidAccountPlanDetails(int customerID, long mobileNo)
			throws CustomerDetailsNotFoundException, PostpaidAccountNotFoundException, PlanDetailsNotFoundException {
			 PostpaidAccount postPaidAccount = getPostPaidAccountDetails(customerID, mobileNo);
		return postPaidAccount.getPlan();
	}
	@Override
	public int createPlan(int monthlyRental, int freeLocalCalls, int freeStdCalls, int freeLocalSMS, int freeStdSMS,
			int freeInternetDataUsageUnits, float localCallRate, float stdCallRate, float localSMSRate,
			float stdSMSRate, float internetDataUsageRate, String planCircle, String planName) {
		Plan plan= new Plan(monthlyRental, freeLocalCalls, freeStdCalls, freeLocalSMS, freeStdSMS, freeInternetDataUsageUnits, localCallRate, stdCallRate, localSMSRate, stdSMSRate, internetDataUsageRate, planCircle, planName);
		planDAO.save(plan);
		return plan.getPlanID();
	}
	@Override
	public Plan getPlanDetails(int planID) throws PlanDetailsNotFoundException {
		return planDAO.findById(planID).orElseThrow(()-> new PlanDetailsNotFoundException());
	}
}
